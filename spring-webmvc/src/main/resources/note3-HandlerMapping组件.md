


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
   
   
##### 3. AbstractHandlerMethodMapping<T>  extends AbstractHandlerMapping implements InitializingBean      
   
   
   ###### 3.1) 属性   
   ```
   private boolean detectHandlerMethodsInAncestorContexts = false;
   
   	private HandlerMethodMappingNamingStrategy<T> namingStrategy;
   
   	private final MappingRegistry mappingRegistry = new MappingRegistry();
   ```
   
   ###### 3.2) 初始化      
   ```
   /**
   	 * Detects handler methods at initialization.
   	 */
   	@Override
   	public void afterPropertiesSet() {
   		initHandlerMethods();
   	}
   ```                         
   ```
   /**
   	 * Scan beans in the ApplicationContext, detect and register handler methods.
   	 * @see #isHandler(Class)
   	 * @see #getMappingForMethod(Method, Class)
   	 * @see #handlerMethodsInitialized(Map)
   	 */
   	protected void initHandlerMethods() {
   	    // 在这里查找requestMapping
   		if (logger.isDebugEnabled()) {
   			logger.debug("Looking for request mappings in application context: " + getApplicationContext());
   		}
   		String[] beanNames = (this.detectHandlerMethodsInAncestorContexts ?
   				BeanFactoryUtils.beanNamesForTypeIncludingAncestors(getApplicationContext(), Object.class) :
   				getApplicationContext().getBeanNamesForType(Object.class));
   
   		for (String beanName : beanNames) {
   			if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
   				Class<?> beanType = null;
   				try {
   					beanType = getApplicationContext().getType(beanName);
   				}
   				catch (Throwable ex) {
   					// An unresolvable bean type, probably from a lazy bean - let's ignore it.
   					if (logger.isDebugEnabled()) {
   						logger.debug("Could not resolve target class for bean with name '" + beanName + "'", ex);
   					}
   				}
   				if (beanType != null && isHandler(beanType)) { // 判断是不是handler    
   					detectHandlerMethods(beanName); // 从handler中查找对应的方法     
   				}
   			}
   		}
   		
   		// 初始化hanlderMathod
   		handlerMethodsInitialized(getHandlerMethods());
   	}
   ```
   
   ```
   /**
   	 * Look for handler methods in a handler.
   	 * @param handler the bean name of a handler or a handler instance
   	 */
   	protected void detectHandlerMethods(final Object handler) {
   		Class<?> handlerType = (handler instanceof String ?
   				getApplicationContext().getType((String) handler) : handler.getClass());
   		final Class<?> userType = ClassUtils.getUserClass(handlerType);
   
   		Map<Method, T> methods = MethodIntrospector.selectMethods(userType,
   				new MethodIntrospector.MetadataLookup<T>() {
   					@Override
   					public T inspect(Method method) {
   						try {
   							return getMappingForMethod(method, userType);
   						}
   						catch (Throwable ex) {
   							throw new IllegalStateException("Invalid mapping on handler class [" +
   									userType.getName() + "]: " + method, ex);
   						}
   					}
   				});
   
   		if (logger.isDebugEnabled()) {
   			logger.debug(methods.size() + " request handler methods found on " + userType + ": " + methods);
   		}
   		for (Map.Entry<Method, T> entry : methods.entrySet()) {
   			Method invocableMethod = AopUtils.selectInvocableMethod(entry.getKey(), userType);
   			T mapping = entry.getValue();
   			registerHandlerMethod(handler, invocableMethod, mapping);
   		}
   	}
   	
   	/**
     * Register a handler method and its unique mapping. Invoked at startup for
     * each detected handler method.
     * @param handler the bean name of the handler or the handler instance
     * @param method the method to register
     * @param mapping the mapping conditions associated with the handler method
     * @throws IllegalStateException if another method was already registered
     * under the same mapping
     */
    protected void registerHandlerMethod(Object handler, Method method, T mapping) {
        this.mappingRegistry.register(mapping, handler, method);
    }
    
    
    /**
     * 初始化handlerMethod时,  先从注册表中获取所有mapping    
     * Return a (read-only) map with all mappings and HandlerMethod's.
     */
    public Map<T, HandlerMethod> getHandlerMethods() {
        this.mappingRegistry.acquireReadLock();
        try {
            return Collections.unmodifiableMap(this.mappingRegistry.getMappings());
        }
        finally {
            this.mappingRegistry.releaseReadLock();
        }
    }
   ```
   
   ```
   /**
   	 * Invoked after all handler methods have been detected.
   	 * @param handlerMethods a read-only map with handler methods and mappings.
   	 */
   	protected void handlerMethodsInitialized(Map<T, HandlerMethod> handlerMethods) {
   	}
   ```
   
   ###### 3.3) 实现getHandlerInternal(request) 查找对应的handlerMethod 
   
   ```
   /**
   	 * Look up a handler method for the given request.
   	 */
   	@Override
   	protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
   		String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
   		if (logger.isDebugEnabled()) {
   			logger.debug("Looking up handler method for path " + lookupPath);
   		}
   		this.mappingRegistry.acquireReadLock();
   		try {
   			HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);  // 通过url查找对应的方法 
   			if (logger.isDebugEnabled()) {
   				if (handlerMethod != null) {
   					logger.debug("Returning handler method [" + handlerMethod + "]");
   				}
   				else {
   					logger.debug("Did not find handler method for [" + lookupPath + "]");
   				}
   			}
   			return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);
   		}
   		finally {
   			this.mappingRegistry.releaseReadLock();
   		}
   	}
   ```
   
   ```
   /**
   	 * Look up the best-matching handler method for the current request.
   	 * If multiple matches are found, the best match is selected.
   	 * @param lookupPath mapping lookup path within the current servlet mapping
   	 * @param request the current request
   	 * @return the best-matching handler method, or {@code null} if no match
   	 * @see #handleMatch(Object, String, HttpServletRequest)
   	 * @see #handleNoMatch(Set, String, HttpServletRequest)
   	 */
   	protected HandlerMethod lookupHandlerMethod(String lookupPath, HttpServletRequest request) throws Exception {
   		List<Match> matches = new ArrayList<Match>();
   		List<T> directPathMatches = this.mappingRegistry.getMappingsByUrl(lookupPath);
   		if (directPathMatches != null) {
   			addMatchingMappings(directPathMatches, matches, request);
   		}
   		if (matches.isEmpty()) {
   			// No choice but to go through all mappings...
   			addMatchingMappings(this.mappingRegistry.getMappings().keySet(), matches, request);
   		}
   
   		if (!matches.isEmpty()) {
   			Comparator<Match> comparator = new MatchComparator(getMappingComparator(request));
   			Collections.sort(matches, comparator);
   			if (logger.isTraceEnabled()) {
   				logger.trace("Found " + matches.size() + " matching mapping(s) for [" +
   						lookupPath + "] : " + matches);
   			}
   			Match bestMatch = matches.get(0);
   			if (matches.size() > 1) {
   				if (CorsUtils.isPreFlightRequest(request)) {
   					return PREFLIGHT_AMBIGUOUS_MATCH;
   				}
   				Match secondBestMatch = matches.get(1);
   				if (comparator.compare(bestMatch, secondBestMatch) == 0) {
   					Method m1 = bestMatch.handlerMethod.getMethod();
   					Method m2 = secondBestMatch.handlerMethod.getMethod();
   					throw new IllegalStateException("Ambiguous handler methods mapped for HTTP path '" +
   							request.getRequestURL() + "': {" + m1 + ", " + m2 + "}");
   				}
   			}
   			request.setAttribute(BEST_MATCHING_HANDLER_ATTRIBUTE, bestMatch.handlerMethod);
   			handleMatch(bestMatch.mapping, lookupPath, request);
   			return bestMatch.handlerMethod;
   		}
   		else {
   			return handleNoMatch(this.mappingRegistry.getMappings().keySet(), lookupPath, request);
   		}
   	}
   ```
   
   ```
   private void addMatchingMappings(Collection<T> mappings, List<Match> matches, HttpServletRequest request) {
   		for (T mapping : mappings) {
   			T match = getMatchingMapping(mapping, request);
   			if (match != null) {
   				matches.add(new Match(match, this.mappingRegistry.getMappings().get(mapping)));
   			}
   		}
   	}
   
   	/**
   	 * Invoked when a matching mapping is found.
   	 * @param mapping the matching mapping
   	 * @param lookupPath mapping lookup path within the current servlet mapping
   	 * @param request the current request
   	 */
   	protected void handleMatch(T mapping, String lookupPath, HttpServletRequest request) {
   		request.setAttribute(HandlerMapping.PATH_WITHIN_HANDLER_MAPPING_ATTRIBUTE, lookupPath);
   	}
   
   	/**
   	 * Invoked when no matching mapping is not found.
   	 * @param mappings all registered mappings
   	 * @param lookupPath mapping lookup path within the current servlet mapping
   	 * @param request the current request
   	 * @throws ServletException in case of errors
   	 */
   	protected HandlerMethod handleNoMatch(Set<T> mappings, String lookupPath, HttpServletRequest request)
   			throws Exception {
   
   		return null;
   	}
   ```
   
   
   ###### 3.4) 抽象模板方法   
   ```
   // Abstract template methods
   
   	/**
   	 * Whether the given type is a handler with handler methods.
   	 * @param beanType the type of the bean being checked
   	 * @return "true" if this a handler type, "false" otherwise.
   	 */
   	protected abstract boolean isHandler(Class<?> beanType);
   
   	/**
   	 * Provide the mapping for a handler method. A method for which no
   	 * mapping can be provided is not a handler method.
   	 * @param method the method to provide a mapping for
   	 * @param handlerType the handler type, possibly a sub-type of the method's
   	 * declaring class
   	 * @return the mapping, or {@code null} if the method is not mapped
   	 */
   	protected abstract T getMappingForMethod(Method method, Class<?> handlerType);
   
   	/**
   	 * Extract and return the URL paths contained in a mapping.
   	 */
   	protected abstract Set<String> getMappingPathPatterns(T mapping);
   
   	/**
   	 * Check if a mapping matches the current request and return a (potentially
   	 * new) mapping with conditions relevant to the current request.
   	 * @param mapping the mapping to get a match for
   	 * @param request the current HTTP servlet request
   	 * @return the match, or {@code null} if the mapping doesn't match
   	 */
   	protected abstract T getMatchingMapping(T mapping, HttpServletRequest request);
   
   	/**
   	 * Return a comparator for sorting matching mappings.
   	 * The returned comparator should sort 'better' matches higher.
   	 * @param request the current request
   	 * @return the comparator (never {@code null})
   	 */
   	protected abstract Comparator<T> getMappingComparator(HttpServletRequest request);

   ``` 
   
   ###### 3.5) 内部类 MappingRegistry 
   包含了所有查找HandlerMethod的映射方式, 如通过mapping(mappingLookup)查找, 通过命名(nameLookup)查找
   
   
   ####### 3.5.1) 属性 
   ```
   private final Map<T, MappingRegistration<T>> registry = new HashMap<T, MappingRegistration<T>>();
   
    private final Map<T, HandlerMethod> mappingLookup = new LinkedHashMap<T, HandlerMethod>();

    private final MultiValueMap<String, T> urlLookup = new LinkedMultiValueMap<String, T>();

    private final Map<String, List<HandlerMethod>> nameLookup =
            new ConcurrentHashMap<String, List<HandlerMethod>>();

    private final Map<HandlerMethod, CorsConfiguration> corsLookup =
            new ConcurrentHashMap<HandlerMethod, CorsConfiguration>();
    
    private final ReentrantReadWriteLock readWriteLock = new ReentrantReadWriteLock();
   ```
   
   ####### 3.5.2) 注册handlerMethod
   ```
   public void register(T mapping, Object handler, Method method) {
        this.readWriteLock.writeLock().lock();
        try {
            HandlerMethod handlerMethod = createHandlerMethod(handler, method);  // 通过handler, method创建HandlerMethod
            assertUniqueMethodMapping(handlerMethod, mapping);

            if (logger.isInfoEnabled()) {
                logger.info("Mapped \"" + mapping + "\" onto " + handlerMethod);
            }
            this.mappingLookup.put(mapping, handlerMethod);  // 通过mappging 注册

            List<String> directUrls = getDirectUrls(mapping);
            for (String url : directUrls) {
                this.urlLookup.add(url, mapping);  // 通过url注册  
            }

            String name = null;
            if (getNamingStrategy() != null) {
                name = getNamingStrategy().getName(handlerMethod, mapping);
                addMappingName(name, handlerMethod);  // 通过mapping名注册 
            }

            CorsConfiguration corsConfig = initCorsConfiguration(handler, method, mapping);
            if (corsConfig != null) {
                this.corsLookup.put(handlerMethod, corsConfig);
            }

            
            // mappingRegistration是所有映射信息
            this.registry.put(mapping, new MappingRegistration<T>(mapping, handlerMethod, directUrls, name));
        }
        finally {
            this.readWriteLock.writeLock().unlock();
        }
    }
   ```
   
   ###### 3.6) HandlerMethod    
   
   主要是封装了处理器中的方法信息, 提供了方便的访问方式, 如方法参数,方法返回值   
   
   #######  3.6.1) 属性    
   ```
   /** Logger that is available to subclasses */
   	protected final Log logger = LogFactory.getLog(getClass());
   
   	private final Object bean;
   
   	private final BeanFactory beanFactory;
   
   	private final Class<?> beanType;
   
   	private final Method method;
   
   	private final Method bridgedMethod;
   
   	private final MethodParameter[] parameters;
   
   	private HttpStatus responseStatus;
   
   	private String responseStatusReason;
   
   	private HandlerMethod resolvedFromHandlerMethod;
   ```
   
   
     
   
   
   
   
   
                   
            
                    
   
   




