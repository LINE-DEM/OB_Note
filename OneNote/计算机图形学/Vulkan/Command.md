指令被录制到CommandBuffer内存上  
然后提交到渲染或者呈现队列  
按顺序提交到GPU 但是有可能CMD3虽然是后发送的 但是先执行完毕了

![Exported image](Exported%20image%2020260112170301-0.png)   
为什么要有多个队列：

![Exported image](Exported%20image%2020260112170302-1.png)