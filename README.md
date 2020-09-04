# LexoAop
---
**This is an unvalidated, unsupported fork.  The following is a translation of the original README.  Again, I have not validated this library - I simply forked it to explore it.  Proceed with caution.**
---
This project is a compile-time AOP framework implemented using mono.cecil.

##### 1. How to use
  &emsp; &emsp;First inherit the base class `MethodAspect` (`MethodAspect` inherits from `Attribute` ), and then use it directly as an `Attribute`.
   The original author used this framework to create [Leox.Transaction](https://github.com/linkypi/Leox.Transaction), which provides a `Transactional` attribute which implements transaction handling.
``` c#
    class Program
    {
        static void Main(string[] args)
        {
          var count = Write("nice,well done.")
		  Console.WriteLine(string.Format("your had write {0} words",count.ToString()));
        }
		
		[Log(Order = 1, ExceptionStrategy = ExceptionStrategy.UnThrow), Timing(Order = 2)]
        public static int Write(string title, string words) {
           Console.WriteLine(string.Format("title: {0}, content: {1}",title,words));
		   return string.IsNullOrEmpty(words)? 0 : words.Length;
        }
    }
	
    class Timing : MethodAspect
    {
        public override void OnStart(MethodAspectArgs args)
        {
		    Console.WriteLine("timing start");
			Console.WriteLine("method args: ");
            if (args != null && args.Argument != null) {
                foreach (var item in args.Argument)
                {
                    Console.Write(item.ToString() + "  ");
                }
            }
        }

        public override void OnEnd(MethodAspectArgs args)
        {
            Console.WriteLine("timing end");
        }

        public override void OnSuccess(MethodAspectArgs args)
        {
            Console.WriteLine("timing success : " + args.ReturnValue.ToString());
        }
		
	    public override void OnException(MethodAspectArgs args)
        {
            Console.WriteLine("timing on exception: " + args.Exception.Message);
        }
    }
	
	class Log : MethodAspect
    {
        public override void OnException(MethodAspectArgs args)
        {
            Console.WriteLine("log on exception: " + args.Exception.Message);
        }
        public override void OnStart(MethodAspectArgs args)
        {
            Console.WriteLine("log start");
        }

        public override void OnEnd(MethodAspectArgs args)
        {
            Console.WriteLine("log end");
        }

        public override void OnSuccess(MethodAspectArgs args)
        {
            Console.WriteLine("log success. ");
			if(args.ReturnValue!= null)
				Console.WriteLine("log success, return value : " + args.ReturnValue.ToString()); 
        }
    }
```

##### 2. Leox.Aop , Aop base class, such as method intercept base class MethodAspect, exception handling strategy ExceptionStrategy

##### 3. Leox.Injector Implementation of injecting IL code,
Generate dll or inject tool class exe by switching the output type in the project properties. The current implementation is method-level interception, the basic idea:
   -Load the assembly and find the method marked with MethodAspect Attribute
   -Copy the method and generate a new method copy_method, the original method is clear after the copy is completed
   -Rewrite the original method, first call the Start method of AopAttribute
   -Execute copy_method, if the method is an instance method, you need to load this from ldarg0, and execute the OnSuccess method if the execution is successful
   -If copy_method executes an error, the exception will be caught. The exception handling provides three processing methods.
     One is to only capture and not throw, which is UnThrow, and the second is to throw this ex, which is ThrowNew,
The last one is ReThrow (default), which is the code ``` c# throw ```, which saves the complete exception tracking chain
If there are multiple AopAttributes that specify the ExceptionStrategy attribute, only the one with the smallest Order will prevail, that is, the one at the top will prevail
There are three ways of exception handling
    ``` c#
	    public enum ExceptionStrategy
		{
			UnThrow,
			/// <summary>
			/// throw ex; 
			/// </summary>
			ThrowNew,
			/// <summary>
			/// throw; //保存有异常跟踪链
			/// </summary>
			ReThrow
		}
    ``` 
  - The above processing is completed and finally call the OnEnd method of AopAttribute
  
  Remember to box Box when transferring value types to reference types.
  If you think IL code is really difficult to write, then use a dumb way, which is to write the corresponding C# code first.
  After the compilation is passed, use ILSpy or ildasm to view the corresponding part of the il code, and then just write it according to the inside.
  There is a MethodCache class in Injector.cs that is useless, because the cache is dead, the version of the module in the next injection
  The guid is different from the version of the last injected module, so injecting the old version of the method into the new module will report an error.
  
  In many cases, some unknown errors will occur after injection, and I don’t know how to debug. I have to return to the c# code I wrote to view the IL instructions line by line.
  This is very painful. So here is an IL-level code debugging tool - dotnet il editor.
  The method of use is:
  -Create a new console project, add a line ``` Console.ReadKey(); ``` before executing the specified method, and then generate an exe, that is, compile and inject
  -Then open the exe, then the process is suspended and you need to enter characters, but you cannot enter at this time
  -Now open the dotnet il editor, drag the exe to the project explorer, find the il code you want to execute and press F9 to break the point
  -Then find Debug -> Attach to process... in this editor and click OK after finding the exe process just opened
  -At this time, return to the exe window and enter a character at random to debug IL
  
The injected code looks like this：
  
   ``` c#
	[Log(Order = 1, ExceptionStrategy = ExceptionStrategy.UnThrow), Timing(Order = 12)]
	public int Say(string words, int age)
	{
		MemberInfo method = typeof(Service).GetMethod("Say", new Type[]
		{
			typeof(string),
			typeof(int)
		});
		MethodAspectArgs methodAspectArgs = new MethodAspectArgs();
		methodAspectArgs.Argument = new object[]
		{
			words,
			age
		};
		MAList mAList = new MAList();
		Log item = method.GetCustomAttributes(typeof(Log), false)[0] as Log;
		mAList.Add(item);
		Timing item2 = method.GetCustomAttributes(typeof(Timing), false)[0] as Timing;
		mAList.Add(item2);
		foreach (MethodAspect current in mAList)
		{
			current.OnStart(methodAspectArgs);
		}
		int num;
		try
		{
			num = this._Say_(words, age);
			methodAspectArgs.ReturnValue = num;
			foreach (MethodAspect current in mAList)
			{
				current.OnSuccess(methodAspectArgs);
			}
		}
		catch (Exception ex)
		{
			methodAspectArgs.Exception = ex;
			foreach (MethodAspect current in mAList)
			{
				current.OnException(methodAspectArgs);
			}
			switch ((int)mAList[0].ExceptionStrategy)
			{
			case 1:
				throw ex;
			case 2:
				throw;
			}
		}
		foreach (MethodAspect current in mAList)
		{
			current.OnEnd(methodAspectArgs);
		}
		return num;
	}
   ```

##### 4. Leox.BuildTask 
&emsp; &emsp; Customize an MSBuild Task, that is, add the following content to the csproj of the used project to achieve the specified project
After generation, execute the Inject Task to achieve the purpose of injection.
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
 To debug the BuildTask project, you need to do the following configuration
  1. Create a new file BuildSample.proj in the project, right-click the file and copy it to the output directory.
  1. Right-click the project to open the properties, select Debug -> Start Operation -> select Start external program,
     Enter C:\Windows\Microsoft.NET\Framework64\v4.0.30319\MSBuild.exe,
     Pay attention to distinguish between 32-bit and 64-bit Framework. If you fill in the wrong way, an error will be reported and debugging
  2. Command line parameter input: BuildSample.proj /fl /flp:v=diag, BuildSample.proj must have
  3. Fill in the full path of bin/Debug corresponding to the project in the working directory
 
##### 5. Leox.AopBuildTest 
&emsp; &emsp;Refer to the BuildTask project to test the Aop project. There is a problem here is that when the BuildTask task is executed, MSBuild will lock the dll used in libs.
   If you want to update, you have to turn off the MSBuild process before you can update. If you think this is too troublesome, you can set it
   The environment variable MSBUILDDISABLENODEREUSE is set to 1 so that the MSBuild process will not stay in memory for a long time
  
##### 6.Remaining problem:
&emsp; &emsp; After the assembly is injected, VS cannot be used to debug. If anyone knows how, please tell the original author, thank you!

&emsp; &emsp; Author's Blog:  [http://blog.magicleox.com/](http://blog.magicleox.com/)
