
今天遇到一个问题，需要在API请求结束时，释放数据库链接，避免连接池被爆掉。


按照以往的经验，需要实现IHttpModule，具体不展开了。
但是实现了IHttpModule后，还得去web.config中增加配置，这有点麻烦了，就想有没有简单的办法。


其实是有的，就是在Global.asax.cs里面定义并实现 Application\_EndRequest 方法，在这个方法里面去释放数据库连接即可，经过测试，确实能达到效果。
但是，为什么方法名必须是Application\_EndRequest ？在这之前真不知道为什么，只知道baidu上是这么说的，也能达到效果。
还好我有一点好奇心，想搞清楚是怎么回事情，就把net framework的源码拉下来(其实源代码在电脑里面已经躺了N年了) 分析了一下，以下是分析结果。


省略掉前面N个调用
第一个需要关注的是 HttpApplicationFactory.cs
从名字就知道，这是HttpApplication的工厂类，大家看看Gloabal.asax.cs 里面，是不是这样定义的



```
public class MvcApplication : System.Web.HttpApplication
{
.....
}

```

两者结合起来看，可以推测，HttpApplicationFactory 是用来获取 MvcApplication 实例的，实际情况也是如此 上代码（来自HttpApplicationFactory）



```
internal class HttpApplicationFactory{
  internal const string applicationFileName = "global.asax"; //看到这里，就知道为什么入口文件是global.asax了，因为这里定义死了
  ...
  private void EnsureInited() {
      if (!_inited) {
          lock (this) {
              if (!_inited) {
                  Init();
                  _inited = true;
              }
          }
      }
  }

  private void Init() {
      if (_customApplication != null)
          return;

      try {
          try {
              _appFilename = GetApplicationFile();

              CompileApplication();
          }
          finally {
              // Always set up global.asax file change notification, even if compilation
              // failed.  This way, if the problem is fixed, the appdomain will be restarted.
              SetupChangesMonitor();
          }
      }
      catch { // Protect against exception filters
          throw;
      }
  }
  private void CompileApplication() {
    // Get the Application Type and AppState from the global file

    _theApplicationType = BuildManager.GetGlobalAsaxType();

    BuildResultCompiledGlobalAsaxType result = BuildManager.GetGlobalAsaxBuildResult();

    if (result != null) {

        // Even if global.asax was already compiled, we need to get the collections
        // of application and session objects, since they are not persisted when
        // global.asax is compiled.  Ideally, they would be, but since  tags
        // are only there for ASP compat, it's not worth the trouble.
        // Note that we only do this is the rare case where we know global.asax contains
        //  tags, to avoid always paying the price (VSWhidbey 453101)
        if (result.HasAppOrSessionObjects) {
            GetAppStateByParsingGlobalAsax();
        }

        // Remember file dependencies
        _fileDependencies = result.VirtualPathDependencies;
    }

    if (_state == null) {
        _state = new HttpApplicationState();
    }


    // Prepare to hookup event handlers via reflection

    ReflectOnApplicationType();
  }   

  private void ReflectOnApplicationType() {
    ArrayList handlers = new ArrayList();
    MethodInfo[] methods;

    Debug.Trace("PipelineRuntime", "ReflectOnApplicationType");

    // get this class methods
    methods = _theApplicationType.GetMethods(BindingFlags.NonPublic | BindingFlags.Public | BindingFlags.Instance | BindingFlags.Static);
    foreach (MethodInfo m in methods) {
        if (ReflectOnMethodInfoIfItLooksLikeEventHandler(m))
            handlers.Add(m);
    }
    
    // get base class private methods (GetMethods would not return those)
    Type baseType = _theApplicationType.BaseType;
    if (baseType != null && baseType != typeof(HttpApplication)) {
        methods = baseType.GetMethods(BindingFlags.NonPublic | BindingFlags.Instance | BindingFlags.Static);
        foreach (MethodInfo m in methods) {
            if (m.IsPrivate && ReflectOnMethodInfoIfItLooksLikeEventHandler(m))
                handlers.Add(m);
        }
    }

    // remember as an array
    _eventHandlerMethods = new MethodInfo[handlers.Count];
    for (int i = 0; i < _eventHandlerMethods.Length; i++)
        _eventHandlerMethods[i] = (MethodInfo)handlers[i];
  }
  
  private bool ReflectOnMethodInfoIfItLooksLikeEventHandler(MethodInfo m) {
    if (m.ReturnType != typeof(void))
        return false;

    // has to have either no args or two args (object, eventargs)
    ParameterInfo[] parameters = m.GetParameters();

    switch (parameters.Length) {
        case 0:
            // ok
            break;
        case 2:
            // param 0 must be object
            if (parameters[0].ParameterType != typeof(System.Object))
                return false;
            // param 1 must be eventargs
            if (parameters[1].ParameterType != typeof(System.EventArgs) &&
                !parameters[1].ParameterType.IsSubclassOf(typeof(System.EventArgs)))
                return false;
            // ok
            break;

        default:
            return false;
    }

    // check the name (has to have _ not as first or last char)
    String name = m.Name;
    int j = name.IndexOf('_');
    if (j <= 0 || j > name.Length-1)
        return false;

    // special pseudo-events
    if (StringUtil.EqualsIgnoreCase(name, "Application_OnStart") ||
        StringUtil.EqualsIgnoreCase(name, "Application_Start")) {
        _onStartMethod = m;
        _onStartParamCount = parameters.Length;
    }
    else if (StringUtil.EqualsIgnoreCase(name, "Application_OnEnd") ||
             StringUtil.EqualsIgnoreCase(name, "Application_End")) {
        _onEndMethod = m;
        _onEndParamCount = parameters.Length;
    }
    else if (StringUtil.EqualsIgnoreCase(name, "Session_OnEnd") ||
             StringUtil.EqualsIgnoreCase(name, "Session_End")) {
        _sessionOnEndMethod = m;
        _sessionOnEndParamCount = parameters.Length;
    }

    return true;
  }
}

```

上面代码调用链路是EnsureInited\-\>Init\-\>CompileApplication\-\>ReflectOnApplicationType\-\>ReflectOnMethodInfoIfItLooksLikeEventHandler ,核心作用是：将MvcApplication中，方法名包含下划线、方法参数为空或者有2个参数(第一个参数的类型是Object,第二个参数的类型是EventArgs) 的方法加入到\_eventHandlerMethods 中
那么事件是怎么绑定的呢？继续上代码



```
internal class HttpApplicationFactory{

......
  using (new ApplicationImpersonationContext()) {
      app.InitInternal(context, _state, _eventHandlerMethods);
  }
......


......
  using (new ApplicationImpersonationContext()) {
      app.InitInternal(context, _state, _eventHandlerMethods);
  }
......
}

```


```
// HttpApplication.cs
public class HttpApplication{
  internal void InitSpecial(HttpApplicationState state, MethodInfo[] handlers, IntPtr appContext, HttpContext context) {
    .....    
      if (handlers != null) {
          HookupEventHandlersForApplicationAndModules(handlers);
      }
    .....
  }
   
  internal void InitInternal(HttpContext context, HttpApplicationState state, MethodInfo[] handlers) {
    .....
      if (handlers != null)        
        HookupEventHandlersForApplicationAndModules(handlers);
    .....
  }
  private void HookupEventHandlersForApplicationAndModules(MethodInfo[] handlers) {
      _currentModuleCollectionKey = HttpApplicationFactory.applicationFileName;

      if(null == _pipelineEventMasks) {
          Dictionary<string, RequestNotification> dict = new Dictionary<string, RequestNotification>();
          BuildEventMaskDictionary(dict);
          if(null == _pipelineEventMasks) {
              _pipelineEventMasks = dict;
          }
      }


      for (int i = 0; i < handlers.Length; i++) {
          MethodInfo appMethod = handlers[i];
          String appMethodName = appMethod.Name;
          int namePosIndex = appMethodName.IndexOf('_');
          String targetName = appMethodName.Substring(0, namePosIndex);

          // Find target for method
          Object target = null;

          if (StringUtil.EqualsIgnoreCase(targetName, "Application"))
              target = this;
          else if (_moduleCollection != null)
              target = _moduleCollection[targetName];

          if (target == null)
              continue;

          // Find event on the module type
          Type targetType = target.GetType();
          EventDescriptorCollection events = TypeDescriptor.GetEvents(targetType);
          string eventName = appMethodName.Substring(namePosIndex+1);

          EventDescriptor foundEvent = events.Find(eventName, true);
          if (foundEvent == null
              && StringUtil.EqualsIgnoreCase(eventName.Substring(0, 2), "on")) {

              eventName = eventName.Substring(2);
              foundEvent = events.Find(eventName, true);
          }

          MethodInfo addMethod = null;
          if (foundEvent != null) {
              EventInfo reflectionEvent = targetType.GetEvent(foundEvent.Name);
              Debug.Assert(reflectionEvent != null);
              if (reflectionEvent != null) {
                  addMethod = reflectionEvent.GetAddMethod();
              }
          }

          if (addMethod == null)
              continue;

          ParameterInfo[] addMethodParams = addMethod.GetParameters();

          if (addMethodParams.Length != 1)
              continue;

          // Create the delegate from app method to pass to AddXXX(handler) method

          Delegate handlerDelegate = null;

          ParameterInfo[] appMethodParams = appMethod.GetParameters();

          if (appMethodParams.Length == 0) {
              // If the app method doesn't have arguments --
              // -- hookup via intermidiate handler

              // only can do it for EventHandler, not strongly typed
              if (addMethodParams[0].ParameterType != typeof(System.EventHandler))
                  continue;

              ArglessEventHandlerProxy proxy = new ArglessEventHandlerProxy(this, appMethod);
              handlerDelegate = proxy.Handler;
          }
          else {
              // Hookup directly to the app methods hoping all types match

              try {
                  handlerDelegate = Delegate.CreateDelegate(addMethodParams[0].ParameterType, this, appMethodName);
              }
              catch {
                  // some type mismatch
                  continue;
              }
          }

          // Call the AddXXX() to hook up the delegate

          try {
              addMethod.Invoke(target, new Object[1]{handlerDelegate});
          }
          catch {
              if (HttpRuntime.UseIntegratedPipeline) {
                  throw;
              }
          }

          if (eventName != null) {
              if (_pipelineEventMasks.ContainsKey(eventName)) {
                  if (!StringUtil.StringStartsWith(eventName, "Post")) {
                      _appRequestNotifications |= _pipelineEventMasks[eventName];
                  }
                  else {
                      _appPostNotifications |= _pipelineEventMasks[eventName];
                  }
              }
          }
      }
  }
}



```

核心方法：HookupEventHandlersForApplicationAndModules，其作用就是将前面获取到的method与HttpApplication的Event进行绑定(前提是方法名是以Application\_开头的)，


后面就是向IIS注册事件通知了，由于看不到IIS源码，具体怎么做的就不知道了。


最后安利一下，还是用net core吧，更加清晰、直观，谁用谁知道。


 本博客参考[wgetcloud全球加速器服务](https://wgetcloud6.org)。转载请注明出处！
