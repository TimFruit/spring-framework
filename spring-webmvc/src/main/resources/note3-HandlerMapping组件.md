


#### HandlerMapping组件

![HandlerMapping组件类图](img/HandlerMapping组件类图.png)

##### 1. HandlerMapping类继承
   
   ApplicationObjectSupport  implements ApplicationContextAware      
             ^   
            | |   
            | |  拥有applicationContext   
            | |   
   WebApplicationObjectSupport implements ServletContextAware 
             ^        
            | |  拥有servletContext属性,方便访问servletContext      
            | |         
   AbstractHandlerMapping implements HandlerMapping, Ordered     
             ^        
            | | 属性, List<Object> interceptors, List<HandlerInterceptor> adaptedInterceptors       
            | | 实现getHandler(HttpServletRequest)方法, 留有空方法getHandlerInternal(request)供子类实现
            | | 主要作用是将request和handler映射起来, 二是将拦截器,处理器包装成处理器执行链返回   
            | | =============================================================================================| |          
            | |                                                                                              | |                
   AbstractHandlerMethodMapping<T>  implements InitializingBean                                       AbstractUrlHandlerMapping           
             ^         
            | |  <T> 该泛型, 包含了request和handlerMethod匹配的条件              
            | |  属性, MappingRegistry mappingRegistry,  
            | |        HandlerMethodMappingNamingStrategy<T> namingStrategy; 
            | |  实现handlerMethod=getHandlerInternal(request)                      
            | |  主要作用是将request和HandlerMethod映射起来, 映射关系存放于handlerMapping中             
            | |  实现afterPropertiesSet()方法,初始化对应的handlerMethod        
   RequestMappingInfoHandlerMapping            
             ^ 
            | |  抽象类, 实现了匹配条件为RequestMappingInfo的大部分方法
            | |  实现了getMappingPathPatterns(RequestMappingInfo info),           
            | |  getMatchingMapping(RequestMappingInfo info, HttpServletRequest request),             
            | |  getMappingComparator(final HttpServletRequest request)                                     
            | |                 
   RequestMappingHandlerMapping implements MatchableHandlerMapping, EmbeddedValueResolverAware               
             ^       
            | |   实例用于@RequestMapping和@Controller             
            | |   重写RequestMappingInfo getMappingForMethod(Method method, Class<?> handlerType)
            | |   创建适用于@RequestMapping的RquestMappingInfo实例    
            | |   RequestMappingInfo createRequestMappingInfo(RequestMapping requestMapping,          
            | |         RequestCondition<?> customCondition)          
                  			        
   
   
##### 2. AbstractHandlerMethodMapping 

   AbstractHandlerMethodMapping封装了拦截器, 留有getHandlerInternal()供子类实现

   ###### 2.1) 属性  
   
   ```
   private Object defaultHandler;
   
   	private UrlPathHelper urlPathHelper = new UrlPathHelper();
   
   	private PathMatcher pathMatcher = new AntPathMatcher();
   
   	private final List<Object> interceptors = new ArrayList<Object>();
   
   	private final List<HandlerInterceptor> adaptedInterceptors = new ArrayList<HandlerInterceptor>
   ```  
   
   ###### 2.2) 初始化
   ```
   /**
   	 * Initializes the interceptors.
   	 * @see #extendInterceptors(java.util.List)
   	 * @see #initInterceptors()
   	 */
   	@Override
   	protected void initApplicationContext() throws BeansException {
   		extendInterceptors(this.interceptors);
   		detectMappedInterceptors(this.adaptedInterceptors);  //从上下文中搜索查询拦截器 
   		initInterceptors();  // 初始化拦截器 
   	}
   ``` 
    
   
   ```
   /**
   	 * Initialize the specified interceptors, checking for {@link MappedInterceptor}s and
   	 * adapting {@link HandlerInterceptor}s and {@link WebRequestInterceptor}s if necessary.
   	 * @see #setInterceptors
   	 * @see #adaptInterceptor
   	 */
   	protected void initInterceptors() {
   		if (!this.interceptors.isEmpty()) {
   			for (int i = 0; i < this.interceptors.size(); i++) {
   				Object interceptor = this.interceptors.get(i);
   				if (interceptor == null) {
   					throw new IllegalArgumentException("Entry number " + i + " in interceptors array is null");
   				}
   				this.adaptedInterceptors.add(adaptInterceptor(interceptor));
   			}
   		}
   	}
   ``` 
   
   ```
   /**
   	 * Adapt the given interceptor object to the {@link HandlerInterceptor} interface.
   	 * <p>By default, the supported interceptor types are {@link HandlerInterceptor}
   	 * and {@link WebRequestInterceptor}. Each given {@link WebRequestInterceptor}
   	 * will be wrapped in a {@link WebRequestHandlerInterceptorAdapter}.
   	 * Can be overridden in subclasses.
   	 * @param interceptor the specified interceptor object
   	 * @return the interceptor wrapped as HandlerInterceptor
   	 * @see org.springframework.web.servlet.HandlerInterceptor
   	 * @see org.springframework.web.context.request.WebRequestInterceptor
   	 * @see WebRequestHandlerInterceptorAdapter
   	 */
   	protected HandlerInterceptor adaptInterceptor(Object interceptor) {
   		if (interceptor instanceof HandlerInterceptor) {
   			return (HandlerInterceptor) interceptor;
   		}
   		else if (interceptor instanceof WebRequestInterceptor) {
   			return new WebRequestHandlerInterceptorAdapter((WebRequestInterceptor) interceptor);
   		}
   		else {
   			throw new IllegalArgumentException("Interceptor type not supported: " + interceptor.getClass().getName());
   		}
   	}
   ```
    
    
   
   ###### 2.3) getHandler(request)   -- 实现HandlerMapping接口, 供dispatcherServlet#doDispatch()调用
   
   ```
   /**
   	 * Look up a handler for the given request, falling back to the default
   	 * handler if no specific one is found.
   	 * @param request current HTTP request
   	 * @return the corresponding handler instance, or the default handler
   	 * @see #getHandlerInternal
   	 */
   	@Override
   	public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
   		Object handler = getHandlerInternal(request);  // 该方法留给子类实现  
   		if (handler == null) {
   			handler = getDefaultHandler();
   		}
   		if (handler == null) {
   			return null;
   		}
   		// Bean name or resolved handler?
   		if (handler instanceof String) {
   			String handlerName = (String) handler;
   			handler = getApplicationContext().getBean(handlerName);
   		}
   
   		HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);
   		if (CorsUtils.isCorsRequest(request)) {
   			CorsConfiguration globalConfig = this.globalCorsConfigSource.getCorsConfiguration(request);
   			CorsConfiguration handlerConfig = getCorsConfiguration(handler, request);
   			CorsConfiguration config = (globalConfig != null ? globalConfig.combine(handlerConfig) : handlerConfig);
   			executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
   		}
   		return executionChain;
   	}
   ```
   
   ```
   /**
   	 * Build a {@link HandlerExecutionChain} for the given handler, including
   	 * applicable interceptors.
   	 * <p>The default implementation builds a standard {@link HandlerExecutionChain}
   	 * with the given handler, the handler mapping's common interceptors, and any
   	 * {@link MappedInterceptor}s matching to the current request URL. Interceptors
   	 * are added in the order they were registered. Subclasses may override this
   	 * in order to extend/rearrange the list of interceptors.
   	 * <p><b>NOTE:</b> The passed-in handler object may be a raw handler or a
   	 * pre-built {@link HandlerExecutionChain}. This method should handle those
   	 * two cases explicitly, either building a new {@link HandlerExecutionChain}
   	 * or extending the existing chain.
   	 * <p>For simply adding an interceptor in a custom subclass, consider calling
   	 * {@code super.getHandlerExecutionChain(handler, request)} and invoking
   	 * {@link HandlerExecutionChain#addInterceptor} on the returned chain object.
   	 * @param handler the resolved handler instance (never {@code null})
   	 * @param request current HTTP request
   	 * @return the HandlerExecutionChain (never {@code null})
   	 * @see #getAdaptedInterceptors()
   	 */
   	protected HandlerExecutionChain getHandlerExecutionChain(Object handler, HttpServletRequest request) {
   		HandlerExecutionChain chain = (handler instanceof HandlerExecutionChain ?
   				(HandlerExecutionChain) handler : new HandlerExecutionChain(handler));
   
   		String lookupPath = this.urlPathHelper.getLookupPathForRequest(request);
   		for (HandlerInterceptor interceptor : this.adaptedInterceptors) { // 这里使用adaptedInterceptors
   			if (interceptor instanceof MappedInterceptor) {
   				MappedInterceptor mappedInterceptor = (MappedInterceptor) interceptor;
   				if (mappedInterceptor.matches(lookupPath, this.pathMatcher)) {  // 判断是否匹配 
   					chain.addInterceptor(mappedInterceptor.getInterceptor());
   				}
   			}
   			else {
   				chain.addInterceptor(interceptor);
   			}
   		}
   		return chain;
   	}
   ```
   
   ###### 2.4) HandlerInterceptor
   ```
   /**
   	 * 在handlerMapping决定方法之后, 在handlerAdapter调用处理方法之前,
   	 * 用于判断是否到此拦截器终止处理, return true 继续执行, return false 到此拦截器终止执行 
   	 */
   	boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
   			throws Exception;
   
   	/**
   	 * 在handlerAdapter调用handle方法之后, 在渲染视图之前, 用于为视图提供额外的model
   	 */
   	void postHandle(
   			HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView)
   			throws Exception;
   
   	/**
   	 * 在渲染视图之后逆序处理, 一般用于清理资源
   	 * 逆序, 第一个拦截器将会在最后一个处理
   	 */
   	void afterCompletion(
   			HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex)
   			throws Exception;
   ```                         
   
   
   
                          
                   
            
                    
   
   




