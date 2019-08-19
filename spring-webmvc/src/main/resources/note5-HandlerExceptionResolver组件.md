

#### HandlerExceptionResolver组件  

   ![HandlerExceptionResolver类图](./img/HandlerExceptionResolver组件类图.png)

##### 1. HandlerExceptionResolver类图     
   
   HandlerExceptionResolver
             ^       
             |    // 将异常转换成一个精确的错误页                
             |    ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex);      
             |
             | -------------------------------------------------------------------------------------------------|
             |                                                                                                  |           
   AbstractHandlerMethodExceptionResolver                                                             DefaultHandlerExceptionResolver                                           
             ^         
            | |     实现resolveException(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex)            
            | |     留有doResolveException(request, response, handler, ex)给子类实现               
            | |         
   ExceptionHandlerExceptionResolver        
            | |                                                      
            | |      实现doResolveHandlerMethodException()方法      
            | |           

##### 2. AbstractHandlerMethodExceptionResolver     
 
   ###### 2.1) 属性 
   ```
   private int order = Ordered.LOWEST_PRECEDENCE;
   
   	private Set<?> mappedHandlers;
   
   	private Class<?>[] mappedHandlerClasses;
   
   	private Log warnLogger;
   
   	private boolean preventResponseCaching = false;
   ```

   ###### 2.2) 实现对应的方法  
   ```
   /**
   	 * Check whether this resolver is supposed to apply (i.e. if the supplied handler
   	 * matches any of the configured {@linkplain #setMappedHandlers handlers} or
   	 * {@linkplain #setMappedHandlerClasses handler classes}), and then delegate
   	 * to the {@link #doResolveException} template method.
   	 */
   	@Override
   	public ModelAndView resolveException(
   			HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
   
   		if (shouldApplyTo(request, handler)) { // 判断本exceptionResolver是否适用于本handler
   			prepareResponse(ex, response);  // 准备response  
   			ModelAndView result = doResolveException(request, response, handler, ex); // 留有模板给子类实现 
   			if (result != null) {
   				// Print debug message when warn logger is not enabled.
   				if (logger.isDebugEnabled() && (this.warnLogger == null || !this.warnLogger.isWarnEnabled())) {
   					logger.debug("Resolved [" + ex + "]" + (result.isEmpty() ? "" : " to " + result));
   				}
   				// Explicitly configured warn logger in logException method.
   				logException(ex, request);
   			}
   			return result;
   		}
   		else {
   			return null;
   		}
   	}
   ```

##### 3. ExceptionHandlerExceptionResolver  extends AbstractHandlerMethodExceptionResolver implements ApplicationContextAware, InitializingBean   

   
   ###### 3.1)  属性  
   ```
   private List<HandlerMethodArgumentResolver> customArgumentResolvers;
   
   	private HandlerMethodArgumentResolverComposite argumentResolvers;
   
   	private List<HandlerMethodReturnValueHandler> customReturnValueHandlers;
   
   	private HandlerMethodReturnValueHandlerComposite returnValueHandlers;
   
   	private List<HttpMessageConverter<?>> messageConverters;
   
   	private ContentNegotiationManager contentNegotiationManager = new ContentNegotiationManager();
   
   	private final List<Object> responseBodyAdvice = new ArrayList<Object>();
   
   	private ApplicationContext applicationContext;
   
   	private final Map<Class<?>, ExceptionHandlerMethodResolver> exceptionHandlerCache =
   			new ConcurrentHashMap<Class<?>, ExceptionHandlerMethodResolver>(64);
   
   	private final Map<ControllerAdviceBean, ExceptionHandlerMethodResolver> exceptionHandlerAdviceCache =
   			new LinkedHashMap<ControllerAdviceBean, ExceptionHandlerMethodResolver>();
   ```

   ###### 3.2) 初始化 
   ```
   public ExceptionHandlerExceptionResolver() {
   		StringHttpMessageConverter stringHttpMessageConverter = new StringHttpMessageConverter();
   		stringHttpMessageConverter.setWriteAcceptCharset(false);  // see SPR-7316
   
   		this.messageConverters = new ArrayList<HttpMessageConverter<?>>();
   		this.messageConverters.add(new ByteArrayHttpMessageConverter());
   		this.messageConverters.add(stringHttpMessageConverter);
   		this.messageConverters.add(new SourceHttpMessageConverter<Source>());
   		this.messageConverters.add(new AllEncompassingFormHttpMessageConverter());
   	}
   ```

   ```
   @Override
   	public void afterPropertiesSet() {
   		// Do this first, it may add ResponseBodyAdvice beans
   		initExceptionHandlerAdviceCache();
   
   		if (this.argumentResolvers == null) {
   			List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers();
   			this.argumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
   		}
   		if (this.returnValueHandlers == null) {
   			List<HandlerMethodReturnValueHandler> handlers = getDefaultReturnValueHandlers();
   			this.returnValueHandlers = new HandlerMethodReturnValueHandlerComposite().addHandlers(handlers);
   		}
   	}
   ```
   
   ```
   private void initExceptionHandlerAdviceCache() {
   		if (getApplicationContext() == null) {
   			return;
   		}
   		if (logger.isDebugEnabled()) {
   			logger.debug("Looking for exception mappings: " + getApplicationContext());
   		}
   
   		List<ControllerAdviceBean> adviceBeans = ControllerAdviceBean.findAnnotatedBeans(getApplicationContext());
   		AnnotationAwareOrderComparator.sort(adviceBeans);
   
   		for (ControllerAdviceBean adviceBean : adviceBeans) {
   			ExceptionHandlerMethodResolver resolver = new ExceptionHandlerMethodResolver(adviceBean.getBeanType());
   			if (resolver.hasExceptionMappings()) {
   				this.exceptionHandlerAdviceCache.put(adviceBean, resolver);
   				if (logger.isInfoEnabled()) {
   					logger.info("Detected @ExceptionHandler methods in " + adviceBean);
   				}
   			}
   			if (ResponseBodyAdvice.class.isAssignableFrom(adviceBean.getBeanType())) {
   				this.responseBodyAdvice.add(adviceBean);
   				if (logger.isInfoEnabled()) {
   					logger.info("Detected ResponseBodyAdvice implementation in " + adviceBean);
   				}
   			}
   		}
   	}
   ```

   ###### 3.3) 实现父类方法  
   ```
   /**
   	 * Find an {@code @ExceptionHandler} method and invoke it to handle the raised exception.
   	 */
   	@Override
   	protected ModelAndView doResolveHandlerMethodException(HttpServletRequest request,
   			HttpServletResponse response, HandlerMethod handlerMethod, Exception exception) {
   
   		ServletInvocableHandlerMethod exceptionHandlerMethod = getExceptionHandlerMethod(handlerMethod, exception);
   		if (exceptionHandlerMethod == null) {
   			return null;
   		}
   
   		exceptionHandlerMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
   		exceptionHandlerMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
   
   		ServletWebRequest webRequest = new ServletWebRequest(request, response);
   		ModelAndViewContainer mavContainer = new ModelAndViewContainer();
   
   		try {
   			if (logger.isDebugEnabled()) {
   				logger.debug("Invoking @ExceptionHandler method: " + exceptionHandlerMethod);
   			}
   			Throwable cause = exception.getCause();
   			if (cause != null) {
   				// Expose cause as provided argument as well
   				exceptionHandlerMethod.invokeAndHandle(webRequest, mavContainer, exception, cause, handlerMethod);
   			}
   			else {
   				// Otherwise, just the given exception as-is
   				exceptionHandlerMethod.invokeAndHandle(webRequest, mavContainer, exception, handlerMethod);
   			}
   		}
   		catch (Throwable invocationEx) {
   			// Any other than the original exception is unintended here,
   			// probably an accident (e.g. failed assertion or the like).
   			if (invocationEx != exception && logger.isWarnEnabled()) {
   				logger.warn("Failed to invoke @ExceptionHandler method: " + exceptionHandlerMethod, invocationEx);
   			}
   			// Continue with default processing of the original exception...
   			return null;
   		}
   
   		if (mavContainer.isRequestHandled()) {
   			return new ModelAndView();
   		}
   		else {
   			ModelMap model = mavContainer.getModel();
   			HttpStatus status = mavContainer.getStatus();
   			ModelAndView mav = new ModelAndView(mavContainer.getViewName(), model, status);
   			mav.setViewName(mavContainer.getViewName());
   			if (!mavContainer.isViewReference()) {
   				mav.setView((View) mavContainer.getView());
   			}
   			if (model instanceof RedirectAttributes) {
   				Map<String, ?> flashAttributes = ((RedirectAttributes) model).getFlashAttributes();
   				RequestContextUtils.getOutputFlashMap(request).putAll(flashAttributes);
   			}
   			return mav;
   		}
   	}
   ```



