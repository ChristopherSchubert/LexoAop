# LexoAop
使用 mono.cecil 实现编译时Aop
- Leox.Aop  使用Aop功能时必须继承里面的基类

- Leox.Injector 注入IL代码的实现，通过在项目属性中切换输出类型来生成dll或者注入工具类exe

- Leox.BuildTask  MSBuild的实现，即在使用到的项目的csproj中加入以下内容来实现生成后执行该Task，达到注入的功能。
``` xml
  <PropertyGroup>
  <MyTaskDirectory>libs\</MyTaskDirectory>
  </PropertyGroup>
  <!--UsingTask中的TaskName一定要对应类名-->
  <UsingTask TaskName="AopBuildTask" AssemblyFile="$(MyTaskDirectory)Leox.BuildTask.dll" />
  <Target Name="BeforeBuild">
  </Target>
  <Target Name="AfterBuild">
    <AopBuildTask OutputFile="$(MSBuildProjectFullPath)" TaskFile="Leox.Injector.exe">
      <Output PropertyName="path" TaskParameter="Paths" />
    </AopBuildTask>
    <Message Text="build path: $(path)" />
  </Target>
```
若要调试该项目请在 项目属性-> 调试 做以下配置
1. 启动操作 -> 选中启动外部程序，输入 C:\Windows\Microsoft.NET\Framework64\v4.0.30319\MSBuild.exe ，
   注意区分32、64位的Framework，如果填错则会报错无法调试
2. 命令行参数输入： BuildSample.proj /fl /flp:v=diag ，BuildSample.proj 必须有
3. 工作目录填写该项目对应的bin/Debug全路径
- Leox.AopBuildTest 测试Aop 
使用的是先继承基类，然后直接以Attribute的方式使用即可
``` c#
    public class Log : MethodAspect
    {
        public override void OnStart()
        {
            Console.WriteLine("log start");
        }

        public override void OnEnd()
        {
            Console.WriteLine("log end");
        }
    }
    class Program
    {
        static void Main(string[] args)
        {
          Say("nice,well done.")
        }
        [Log]
        public static void Say(string words) {
           Console.WriteLine(words);
        }
    }
  ```
