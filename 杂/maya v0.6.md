import maya.cmds as cmds
import maya.mel as mel
import os

# ==========================================
# 模块 A: 工具函数库
# ==========================================
def get_all_files_recursive(directory, extensions=[".ma", ".mb"]):
    file_list = []
    for root, dirs, files in os.walk(directory):
        for file in files:
            if os.path.splitext(file)[1].lower() in extensions:
                full_path = os.path.join(root, file).replace("\\", "/")
                file_list.append(full_path)
    return file_list

def fix_broken_references(asset_library_path):
    if not asset_library_path or not os.path.exists(asset_library_path): return
    try:
        ref_nodes = cmds.ls(type="reference")
        for ref_node in ref_nodes:
            if "sharedReferenceNode" in ref_node: continue 
            try:
                old_path = cmds.referenceQuery(ref_node, filename=True)
                filename = os.path.basename(old_path)
                new_path = os.path.join(asset_library_path, filename).replace("\\", "/")
                if os.path.exists(new_path) and new_path != old_path:
                    print(f"--- [引用修复] {filename} -> {new_path}")
                    cmds.file(new_path, loadReference=ref_node)
            except: pass
    except Exception as e:
        print(f"引用检查警告: {e}")

def import_all_references():
    max_recursion = 10
    count = 0
    while count < max_recursion:
        refs = cmds.file(query=True, reference=True)
        if not refs: break
        try:
            cmds.file(refs[0], importReference=True)
        except: break
        count += 1

def remove_all_namespaces():
    cmds.namespace(setNamespace=':')
    namespaces = cmds.namespaceInfo(listOnlyNamespaces=True, recurse=True)
    if namespaces:
        namespaces.sort(key=len, reverse=True)
        for ns in namespaces:
            if ns not in ["UI", "shared"]:
                try:
                    cmds.namespace(moveNamespace=[ns, ":"], force=True)
                    cmds.namespace(removeNamespace=ns)
                except: pass

def clean_history():
    # 1. 清理插件垃圾节点
    unknown_nodes = cmds.ls(type="unknown")
    if unknown_nodes:
        for node in unknown_nodes:
            if cmds.objExists(node):
                try:
                    cmds.lockNode(node, lock=False)
                    cmds.delete(node)
                except: pass
    
    # 2. 清理历史 (注意：不要删除动画曲线)
    # 这里的 constructionHistory=True 通常是安全的，不会删除 AnimCurve
    cmds.delete(all=True, constructionHistory=True)

# ==========================================
# 模块 B: SSC 核心修复
# ==========================================
def run_ssc_fix():
    cmds.cycleCheck(e=False)
    all_joints = cmds.ls(type="joint", long=True)
    if not all_joints: return
    affected_joints = [j for j in all_joints if cmds.getAttr(f"{j}.segmentScaleCompensate")]
    if not affected_joints: return

    print(f"正在修复 {len(affected_joints)} 个骨骼的 SSC...")
    start_time = cmds.playbackOptions(query=True, minTime=True)
    end_time = cmds.playbackOptions(query=True, maxTime=True)
    
    temp_locators = []
    
    # 记录
    for jnt in affected_joints:
        loc = cmds.spaceLocator()[0]
        temp_locators.append(loc)
        pc = cmds.parentConstraint(jnt, loc)[0]
        sc = cmds.scaleConstraint(jnt, loc)[0]
        cmds.bakeResults(loc, time=(start_time, end_time), simulation=True)
        cmds.delete(pc, sc)

    # 应用
    for i, jnt in enumerate(affected_joints):
        loc = temp_locators[i]
        for axis in ['X', 'Y', 'Z']:
            attr = f"{jnt}.scale{axis}"
            if cmds.getAttr(attr, lock=True): cmds.setAttr(attr, lock=False)
            inputs = cmds.listConnections(attr, plugs=True, source=True, destination=False)
            if inputs: cmds.disconnectAttr(inputs[0], attr)
        cmds.setAttr(f"{jnt}.segmentScaleCompensate", 0)
        cmds.scaleConstraint(loc, jnt)

    # 最终烘焙
    cmds.bakeResults(affected_joints, attribute=["scaleX", "scaleY", "scaleZ"], 
                     time=(start_time, end_time), simulation=True, minimizeRotation=True)
    cmds.delete(temp_locators)
    cmds.cycleCheck(e=True)

# ==========================================
# 模块 C: 精准导出逻辑 (v6.0 修正版)
# ==========================================
def export_fbx_custom(output_path):
    start_frame = cmds.playbackOptions(query=True, minTime=True)
    end_frame = cmds.playbackOptions(query=True, maxTime=True)
    
    # --- 1. 智能选择逻辑 (Fix: 必须选中子层级) ---
    targets = ["*DeformationSystem", "*Geometry"]
    
    # 必须清除之前的选择
    cmds.select(clear=True)
    
    found_any = False
    
    for t in targets:
        # 查找节点
        nodes = cmds.ls(t, type="transform", long=True)
        if nodes:
            found_any = True
            # 1. 先选中这个组
            cmds.select(nodes, add=True)
            # 2. 【关键修正】: 选中该组下的所有层级 (Hierarchy)
            # 这样 FBX 烘焙时才会遍历到内部的 Joints
            cmds.select(hierarchy=True)
            
    if not found_any:
        print(f"!!! 警告: 未找到指定组，尝试导出全部。")
        export_mode = True # Export All
    else:
        print(f"已选中导出对象及其层级。")
        export_mode = False # Export Selected

    print(f"配置 FBX 导出: 范围 {start_frame} - {end_frame}")

    # --- 2. FBX 设置 ---
    mel.eval("FBXResetExport")
    
    # 基础
    mel.eval("FBXExportAnimationOnly -v false")
    mel.eval("FBXExportShapes -v true")
    mel.eval("FBXExportSkins -v true")
    mel.eval("FBXExportInputConnections -v false") # 不导出输入连接，只导出烘焙后的结果

    # 烘焙设置
    mel.eval("FBXExportBakeComplexAnimation -v true")
    mel.eval(f"FBXExportBakeComplexStart -v {start_frame}")
    mel.eval(f"FBXExportBakeComplexEnd -v {end_frame}")
    mel.eval("FBXExportBakeComplexStep -v 1")

    # 坐标与语法修正
    mel.eval("FBXExportUpAxis y")       
    mel.eval("FBXExportScaleFactor 1.0") 
    mel.eval("FBXExportInAscii -v false")

    # --- 3. 执行导出 ---
    if export_mode:
        cmds.file(output_path, force=True, options="v=0;", type="FBX export", exportAll=True)
    else:
        # Export Selected (此时 Selection 包含了所有子骨骼)
        cmds.file(output_path, force=True, options="v=0;", type="FBX export", exportSelected=True)

# ==========================================
# 模块 D: 主流水线
# ==========================================
def batch_pipeline_v6():
    if not cmds.pluginInfo("fbxmaya", query=True, loaded=True): cmds.loadPlugin("fbxmaya")

    src_dir = cmds.fileDialog2(fileMode=3, caption="1/3 选择 Maya 源文件目录")[0]
    if not src_dir: return
    
    ref_input = cmds.fileDialog2(fileMode=3, caption="2/3 (可选) 选择引用 Rig 所在的统一文件夹")
    ref_dir = ref_input[0] if ref_input else None
    
    out_dir = cmds.fileDialog2(fileMode=3, caption="3/3 选择 FBX 导出目录")[0]
    if not out_dir: return

    all_files = get_all_files_recursive(src_dir)
    if not all_files: return

    if cmds.confirmDialog(title="开始", message=f"即将处理 {len(all_files)} 个文件。\nv6.0 修正：强制选中层级防止动画丢失", button=["Go", "Cancel"]) == "Cancel": return

    print("=== 流水线启动 ===")
    failed_files = []

    for f in all_files:
        try:
            print(f"\nProcessing: {f}")
            cmds.file(f, open=True, force=True)
            
            if ref_dir: fix_broken_references(ref_dir)
            import_all_references()
            clean_history() 
            run_ssc_fix()
            remove_all_namespaces()
            clean_history()
            
            rel_path = os.path.relpath(f, src_dir)
            final_out_dir = os.path.join(out_dir, os.path.dirname(rel_path))
            if not os.path.exists(final_out_dir): os.makedirs(final_out_dir)
            
            fbx_name = os.path.splitext(os.path.basename(f))[0] + ".fbx"
            out_path = os.path.join(final_out_dir, fbx_name).replace("\\", "/")

            export_fbx_custom(out_path)
            print("导出成功!")

        except Exception as e:
            print(f"!!! 失败: {f}\n{e}")
            failed_files.append(f)
            import traceback
            traceback.print_exc()

    cmds.confirmDialog(title="完成", message=f"全部完成！失败: {len(failed_files)}")

batch_pipeline_v6()