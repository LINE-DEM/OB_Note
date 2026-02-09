import maya.cmds as cmds

def fix_segment_scale_compensate_force():
    """
    v2.0 强力版：自动修复 SSC 问题。
    新增功能：自动解锁属性、断开已有连接，防止报错。
    """
    
    # 1. 获取当前选中的骨骼
    selection = cmds.ls(selection=True, type="joint", long=True)
    
    if not selection:
        cmds.confirmDialog(title="错误", message="请先选择骨骼根节点（Root Joint）。", button=["确定"])
        return

    # 获取时间轴范围
    start_time = cmds.playbackOptions(query=True, minTime=True)
    end_time = cmds.playbackOptions(query=True, maxTime=True)

    cmds.undoInfo(openChunk=True)
    
    try:
        affected_joints = []
        
        # 2. 筛选出所有开启了 SSC 的骨骼
        all_descendants = cmds.listRelatives(selection, allDescendents=True, type="joint", fullPath=True) or []
        target_joints = selection + all_descendants
        
        for jnt in target_joints:
            if cmds.getAttr(f"{jnt}.segmentScaleCompensate"):
                affected_joints.append(jnt)
        
        if not affected_joints:
            cmds.warning("未发现任何开启 Segment Scale Compensate 的骨骼。")
            return

        print(f"检测到 {len(affected_joints)} 个骨骼需要修复 SSC...")

        # 3. 创建视觉参考 (Locators)
        temp_locators = []
        
        for jnt in affected_joints:
            loc = cmds.spaceLocator(name=f"{jnt}_SSC_Ref")[0]
            temp_locators.append(loc)
            
            # 约束 Locator 记录正确位置
            pc = cmds.parentConstraint(jnt, loc, maintainOffset=False)[0]
            sc = cmds.scaleConstraint(jnt, loc, maintainOffset=False)[0]
            
            # 烘焙 Locator
            cmds.bakeResults(loc, time=(start_time, end_time), simulation=True)
            cmds.delete(pc, sc)

        # 4. 核心操作：强制清理、关闭 SSC 并反向约束
        for i, jnt in enumerate(affected_joints):
            loc = temp_locators[i]
            
            # --- v2.0 新增：强制解锁和断开连接 ---
            for axis in ['X', 'Y', 'Z']:
                attr = f"{jnt}.scale{axis}"
                
                # A. 如果属性被锁，强制解锁
                if cmds.getAttr(attr, lock=True):
                    print(f"解锁属性: {attr}")
                    cmds.setAttr(attr, lock=False)
                
                # B. 如果属性已有连接（比如原本的绑定），强制断开
                inputs = cmds.listConnections(attr, plugs=True, source=True, destination=False)
                if inputs:
                    print(f"断开旧连接: {inputs[0]} -> {attr}")
                    cmds.disconnectAttr(inputs[0], attr)
            # ---------------------------------------

            # 关闭 SSC
            cmds.setAttr(f"{jnt}.segmentScaleCompensate", 0)
            
            # 约束回去
            cmds.scaleConstraint(loc, jnt, maintainOffset=False)

        # 5. 再次烘焙
        print("正在烘焙最终数据（可能需要几分钟）...")
        cmds.bakeResults(affected_joints, 
                         attribute=["scaleX", "scaleY", "scaleZ"], 
                         time=(start_time, end_time), 
                         simulation=True,
                         minimizeRotation=True)

        # 6. 清理
        cmds.delete(temp_locators)

        cmds.confirmDialog(title="成功", message=f"成功强力修复了 {len(affected_joints)} 个骨骼！\n请检查动画并导出。", button=["棒！"])

    except Exception as e:
        import traceback
        traceback.print_exc()
        cmds.error(f"脚本依然报错: {e}")
    finally:
        cmds.undoInfo(closeChunk=True)

# 执行函数
fix_segment_scale_compensate_force()