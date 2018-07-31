#### Activity异常情况下的生命周期

&#8195;&#8195;onStop()之前调用onSaveInstante()保存数据，在onCreate之后调用onRestoreInstanceState(Bundle)恢复数据，委托Window以及上层的View保存数据</br>
**内存不足Activity被杀死**</br>
**配置变化导致Activity重建**</br>
配置```android:configChanges=""```可以让配置改变的时候不重启Activity

#### 隐式Intent匹配
**action: **只要指定就必须至少有一个，有一个匹配成功即匹配成功</br>
**category: **可以没有，一旦指定，必须完全匹配</br>
**data: **和action的匹配规则一样

#### Activity的启动模式
**stardard** 标准模式，每次启动均创建一个新的Activity实例;</br>
**singleTop** 栈顶复用，要启动的Activity在栈顶时复用Activity实例，否则和标准模式一样；</br>
**singleTask** 栈内复用，在要启动的任务栈中有Activity的实例就复用；</br>
**singleInstance** 单例模式，独享任务栈</br>

 指定要启动的Activity的任务栈，taskAffinity，默认和包名一样;

 指定启动模式的方式，manifest文件或者Intent中设置flag，不完全一直，可以指定NEW_TASK，SINGLE_TOP，CLEAR_TOP，EXECULE_FROM_RECENT

 [Activity与启动方式详解](http://blog.csdn.net/singwhatiwanna/article/details/9294285)