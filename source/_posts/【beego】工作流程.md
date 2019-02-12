---
title: 【beego】工作流程
date: 2018-11-05 15:11:52
tags:
  - golang
  - beego
---

## 请求流程

app.go => `init()` // 初始化 BeeApp， 创建 ControllerRegister

`ControllerRegister`[/github.com/astaxie/beego/router.go:125]

可以看出 App 结构体的组成: `{
	Handlers *ControllerRegister
	Server *http.Server
}`
在 app.go:104 看到，将 app.Server.Handler 赋值为 app.Handlers

那么，这边有人问了，这个 Handler 要干嘛用的呢？说起 Handler 的作用，就要扯一下 go 里面的 http 请求流程，

请求开始处理的地方：

`func (c \*conn) serve(ctx context.Context)`[/net/http/server.go:1739] -> `serverHandler{c.server}.ServeHTTP(w, w.req)`

`func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request)` -> `handler.ServeHTTP(rw, req)`

这里 `Handler.ServeHTTP` 是一个接口， 而 `ControllerRegister` 也实现了该接口，我们再去看看具体实现，记录几个比较有意思的地方：


```
// Implement http.Handler interface.
func (p *ControllerRegister) ServeHTTP(rw http.ResponseWriter, r *http.Request) {
	startTime := time.Now()
	var (
		runRouter    reflect.Type
		findRouter   bool
		runMethod    string
		methodParams []*param.MethodParam
		routerInfo   *ControllerInfo
		isRunnable   bool
	)
	// :star:✨✨✨ 这里用到了一个pool，用来存取 `beecontext.Context` ，用完又Put放回去了。回头详细了解下 pool 的作用
	context := p.pool.Get().(*beecontext.Context)
	// ✨✨✨ 设置context，关联Request.Context = context
	context.Reset(rw, r)

	defer p.pool.Put(context)
	if BConfig.RecoverFunc != nil {
		// ✨✨✨ 具体实现：/github.com/astaxie/beego/config.go: 204[newBConfig],161[recoverPanic] 
		defer BConfig.RecoverFunc(context)
	}

	...
	...
	...

	// ✨✨✨ 静态文件过滤器
	// filter for static file
	if len(p.filters[BeforeStatic]) > 0 && p.execFilter(context, urlPath, BeforeStatic) {
		goto Admin
	}
	// ✨✨✨ 根据请求uri查找静态文件，比如：favicon.ico，robots.txt
	serverStaticRouter(context)
	// ✨✨✨ 当路由到静态文件时，调用 `ResponseWriter.write` 将 Started=true，即当前请求处理完成
	if context.ResponseWriter.Started {
		findRouter = true
		goto Admin
	}

	if r.Method != http.MethodGet && r.Method != http.MethodHead {
		if BConfig.CopyRequestBody && !context.Input.IsUpload() {
			context.Input.CopyBody(BConfig.MaxMemory)
		}
		context.Input.ParseFormOrMulitForm(BConfig.MaxMemory)
	}

	// ✨✨✨ 这里开始处理Session了，不过我们是接口服务，一般可以把session关闭，即 `SessionOn=False`
	// session init
	if BConfig.WebConfig.Session.SessionOn {
		var err error
		context.Input.CruSession, err = GlobalSessions.SessionStart(rw, r)
		if err != nil {
			logs.Error(err)
			exception("503", context)
			goto Admin
		}
		defer func() {
			if context.Input.CruSession != nil {
				context.Input.CruSession.SessionRelease(rw)
			}
		}()
	}

	// ✨✨✨ 过滤器，这个过滤器filters的实现结构很有意思，结构是个二元数组，根据下标存放不同功能的filter数组
	// ✨✨✨ 而这个下标正好用到了不常见的`iota`
	if len(p.filters[BeforeRouter]) > 0 && p.execFilter(context, urlPath, BeforeRouter) {
		goto Admin
	}
	// ✨✨✨ Input 怎么用的？还得深究一下
	// User can define RunController and RunMethod in filter
	if context.Input.RunController != nil && context.Input.RunMethod != "" {
		findRouter = true
		runMethod = context.Input.RunMethod
		runRouter = context.Input.RunController
	} else {
		// ✨✨✨ 查找路由
		routerInfo, findRouter = p.FindRouter(context)
	}

	//if no matches to url, throw a not found exception
	if !findRouter {
		exception("404", context)
		goto Admin
	}
	if splat := context.Input.Param(":splat"); splat != "" {
		for k, v := range strings.Split(splat, "/") {
			context.Input.SetParam(strconv.Itoa(k), v)
		}
	}

	// ✨✨✨ ？？？
	//execute middleware filters
	if len(p.filters[BeforeExec]) > 0 && p.execFilter(context, urlPath, BeforeExec) {
		goto Admin
	}

	//check policies
	if p.execPolicy(context, urlPath) {
		goto Admin
	}

	if routerInfo != nil {
		//store router pattern into context
		context.Input.SetData("RouterPattern", routerInfo.pattern)
		if routerInfo.routerType == routerTypeRESTFul {
			if _, ok := routerInfo.methods[r.Method]; ok {
				isRunnable = true
				routerInfo.runFunction(context)
			} else {
				exception("405", context)
				goto Admin
			}
		} else if routerInfo.routerType == routerTypeHandler {
			isRunnable = true
			routerInfo.handler.ServeHTTP(rw, r)
		} else {
			runRouter = routerInfo.controllerType
			methodParams = routerInfo.methodParams
			method := r.Method
			if r.Method == http.MethodPost && context.Input.Query("_method") == http.MethodPost {
				method = http.MethodPut
			}
			if r.Method == http.MethodPost && context.Input.Query("_method") == http.MethodDelete {
				method = http.MethodDelete
			}
			if m, ok := routerInfo.methods[method]; ok {
				runMethod = m
			} else if m, ok = routerInfo.methods["*"]; ok {
				runMethod = m
			} else {
				runMethod = method
			}
		}
	}

	// also defined runRouter & runMethod from filter
	if !isRunnable {
		//Invoke the request handler
		var execController ControllerInterface
		if routerInfo != nil && routerInfo.initialize != nil {
			// ✨✨✨ 反射获取到可执行的 `controller`
			execController = routerInfo.initialize()
		} else {
			vc := reflect.New(runRouter)
			var ok bool
			execController, ok = vc.Interface().(ControllerInterface)
			if !ok {
				panic("controller is not ControllerInterface")
			}
		}

		// ✨✨✨ beego.controller 生命周期之 Init
		//call the controller init function
		execController.Init(context, runRouter.Name(), runMethod, execController)

		// ✨✨✨ beego.controller 生命周期之 Prepare
		//call prepare function
		execController.Prepare()

		//if XSRF is Enable then check cookie where there has any cookie in the  request's cookie _csrf
		if BConfig.WebConfig.EnableXSRF {
			execController.XSRFToken()
			if r.Method == http.MethodPost || r.Method == http.MethodDelete || r.Method == http.MethodPut ||
				(r.Method == http.MethodPost && (context.Input.Query("_method") == http.MethodDelete || context.Input.Query("_method") == http.MethodPut)) {
				execController.CheckXSRFCookie()
			}
		}

		execController.URLMapping()

		if !context.ResponseWriter.Started {
			//exec main logic
			switch runMethod {
			case http.MethodGet:
				execController.Get()
			case http.MethodPost:
				execController.Post()
			case http.MethodDelete:
				execController.Delete()
			case http.MethodPut:
				execController.Put()
			case http.MethodHead:
				execController.Head()
			case http.MethodPatch:
				execController.Patch()
			case http.MethodOptions:
				execController.Options()
			default:
				if !execController.HandlerFunc(runMethod) {
					vc := reflect.ValueOf(execController)
					method := vc.MethodByName(runMethod)
					in := param.ConvertParams(methodParams, method.Type(), context)
					out := method.Call(in)

					//For backward compatibility we only handle response if we had incoming methodParams
					if methodParams != nil {
						p.handleParamResponse(context, execController, out)
					}
				}
			}

			//render template
			if !context.ResponseWriter.Started && context.Output.Status == 0 {
				if BConfig.WebConfig.AutoRender {
					if err := execController.Render(); err != nil {
						logs.Error(err)
					}
				}
			}
		}

		// ✨✨✨ beego.controller 生命周期之 Finish
		// finish all runRouter. release resource
		execController.Finish()
	}

	//execute middleware filters
	if len(p.filters[AfterExec]) > 0 && p.execFilter(context, urlPath, AfterExec) {
		goto Admin
	}

	if len(p.filters[FinishRouter]) > 0 && p.execFilter(context, urlPath, FinishRouter) {
		goto Admin
	}

Admin:
	// ✨✨✨ 记录执行信息
	//admin module record QPS

	statusCode := context.ResponseWriter.Status
	if statusCode == 0 {
		statusCode = 200
	}

	logAccess(context, &startTime, statusCode)

	timeDur := time.Since(startTime)
	context.ResponseWriter.Elapsed = timeDur
	if BConfig.Listen.EnableAdmin {
		pattern := ""
		if routerInfo != nil {
			pattern = routerInfo.pattern
		}

		if FilterMonitorFunc(r.Method, r.URL.Path, timeDur, pattern, statusCode) {
			if runRouter != nil {
				go toolbox.StatisticsMap.AddStatistics(r.Method, r.URL.Path, runRouter.Name(), timeDur)
			} else {
				go toolbox.StatisticsMap.AddStatistics(r.Method, r.URL.Path, "", timeDur)
			}
		}
	}

	if BConfig.RunMode == DEV && !BConfig.Log.AccessLogs {
		var devInfo string
		iswin := (runtime.GOOS == "windows")
		statusColor := logs.ColorByStatus(iswin, statusCode)
		methodColor := logs.ColorByMethod(iswin, r.Method)
		resetColor := logs.ColorByMethod(iswin, "")
		if findRouter {
			if routerInfo != nil {
				devInfo = fmt.Sprintf("|%15s|%s %3d %s|%13s|%8s|%s %-7s %s %-3s   r:%s", context.Input.IP(), statusColor, statusCode,
					resetColor, timeDur.String(), "match", methodColor, r.Method, resetColor, r.URL.Path,
					routerInfo.pattern)
			} else {
				devInfo = fmt.Sprintf("|%15s|%s %3d %s|%13s|%8s|%s %-7s %s %-3s", context.Input.IP(), statusColor, statusCode, resetColor,
					timeDur.String(), "match", methodColor, r.Method, resetColor, r.URL.Path)
			}
		} else {
			devInfo = fmt.Sprintf("|%15s|%s %3d %s|%13s|%8s|%s %-7s %s %-3s", context.Input.IP(), statusColor, statusCode, resetColor,
				timeDur.String(), "nomatch", methodColor, r.Method, resetColor, r.URL.Path)
		}
		if iswin {
			logs.W32Debug(devInfo)
		} else {
			logs.Debug(devInfo)
		}
	}
	// Call WriteHeader if status code has been set changed
	if context.Output.Status != 0 {
		context.ResponseWriter.WriteHeader(context.Output.Status)
	}
}
```
