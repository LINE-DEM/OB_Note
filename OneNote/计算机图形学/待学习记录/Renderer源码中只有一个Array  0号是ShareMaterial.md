1.material  
2.GetMaterial(*r)  
3 r-\>GetAndAssignInstantiatedMaterial(==0==, ==false==);  
4 获取默认的ShareMaterial material = GetMaterial(0);  
如果这时候传入比0大的数 就会创建 instantiated 并且放入Array后SetDirty  
5 如果有创建instantiat 走  
GetInstantiatedMaterial 

- ==sharedMaterial 的 get 最终调用来自于 Renderer::GetMaterial 函数== \> 来自 \<[https://blog.csdn.net/Jaihk662/article/details/129077290?app_version=6.4.1&code=app_1562916241&csdn_share_tail=%7B%22type%22%3A%22blog%22%2C%22rType%22%3A%22article%22%2C%22rId%22%3A%22129077290%22%2C%22source%22%3A%222401_86563089%22%7D&uLinkId=usr1mkqgl919blen&utm_source=app](https://blog.csdn.net/Jaihk662/article/details/129077290?app_version=6.4.1&code=app_1562916241&csdn_share_tail=%7B%22type%22%3A%22blog%22%2C%22rType%22%3A%22article%22%2C%22rId%22%3A%22129077290%22%2C%22source%22%3A%222401_86563089%22%7D&uLinkId=usr1mkqgl919blen&utm_source=app)\>  
 \> 来自 \<[https://blog.csdn.net/Jaihk662/article/details/129077290?app_version=6.4.1&code=app_1562916241&csdn_share_tail=%7B%22type%22%3A%22blog%22%2C%22rType%22%3A%22article%22%2C%22rId%22%3A%22129077290%22%2C%22source%22%3A%222401_86563089%22%7D&uLinkId=usr1mkqgl919blen&utm_source=app](https://blog.csdn.net/Jaihk662/article/details/129077290?app_version=6.4.1&code=app_1562916241&csdn_share_tail=%7B%22type%22%3A%22blog%22%2C%22rType%22%3A%22article%22%2C%22rId%22%3A%22129077290%22%2C%22source%22%3A%222401_86563089%22%7D&uLinkId=usr1mkqgl919blen&utm_source=app)\>  

Renderer源码中只有一个Array 0号是ShareMaterial