

#### ViewResolver组件     

   ![ViewResolver组件类图](./img/ViewResolver组件类图.png)

##### 1. ViewResolver组件类图   

   ViewResolver
          ^     
          |    
          | View resolveViewName(String viewName, Locale locale) throws Exception;
          |      
          | ----------------------------------------------------------------------------|       
          |                                                                             |        
   AbstractCachingViewResolver                                                     ResourceBundleViewResolver(解决多国语言)                
          ^     
         | |    
         | |  实现了缓存功能, 用于缓存模板, 子类实现loadView()用于建立一个View对象           
         | |    
   UrlBasedViewResolver       
          ^      
         | |    
         | |  viewName是url的解决方案, redirect:myAction, forward:myAction        
         | |    
   AbstractTemplateViewResolver    
          ^      
         | |    
         | |  模板视图的基类解析器, 一般用于freemarker, velocity                      
         | |    
   FreeMarkerViewResolver        


##### 2. AbstractCachingViewResolver extends WebApplicationObjectSupport implements ViewResolver      

   ###### 2.1) 属性   
   ```
   /** Default maximum number of entries for the view cache: 1024 */
   	public static final int DEFAULT_CACHE_LIMIT = 1024;
   
   	/** Dummy marker object for unresolved views in the cache Maps */
   	private static final View UNRESOLVED_VIEW = new View() {
   		@Override
   		public String getContentType() {
   			return null;
   		}
   		@Override
   		public void render(Map<String, ?> model, HttpServletRequest request, HttpServletResponse response) {
   		}
   	};
   
   
   	/** The maximum number of entries in the cache */
   	private volatile int cacheLimit = DEFAULT_CACHE_LIMIT;
   
   	/** Whether we should refrain from resolving views again if unresolved once */
   	private boolean cacheUnresolved = true;
   
   // 缓存view   
   	/** Fast access cache for Views, returning already cached instances without a global lock */
   	private final Map<Object, View> viewAccessCache = new ConcurrentHashMap<Object, View>(DEFAULT_CACHE_LIMIT);
   
   
   // 使用LRU缓存View实例 
   	/** Map from view key to View instance, synchronized for View creation */
   	@SuppressWarnings("serial")
   	private final Map<Object, View> viewCreationCache =
   			new LinkedHashMap<Object, View>(DEFAULT_CACHE_LIMIT, 0.75f, true) {
   				@Override
   				protected boolean removeEldestEntry(Map.Entry<Object, View> eldest) {
   					if (size() > getCacheLimit()) {
   						viewAccessCache.remove(eldest.getKey());
   						return true;
   					}
   					else {
   						return false;
   					}
   				}
   			};
   ```


   ###### 2.2) 实现接口  
   ```
   @Override
   	public View resolveViewName(String viewName, Locale locale) throws Exception {
   		if (!isCache()) {  // 判断缓存空间大小能否继续缓存   
   			return createView(viewName, locale); // 创建视图   
   		}
   		else {
   			Object cacheKey = getCacheKey(viewName, locale);     
   			View view = this.viewAccessCache.get(cacheKey);  // viewName + "_" + locale
   			if (view == null) {
   				synchronized (this.viewCreationCache) { //全局锁  
   					view = this.viewCreationCache.get(cacheKey);
   					if (view == null) {
   						// Ask the subclass to create the View object.
   						view = createView(viewName, locale);
   						if (view == null && this.cacheUnresolved) {
   							view = UNRESOLVED_VIEW;
   						}
   						if (view != null) {
   							this.viewAccessCache.put(cacheKey, view);  // 存放进去  
   							this.viewCreationCache.put(cacheKey, view);
   							if (logger.isTraceEnabled()) {
   								logger.trace("Cached view [" + cacheKey + "]");
   							}
   						}
   					}
   				}
   			}
   			return (view != UNRESOLVED_VIEW ? view : null);
   		}
   	}
   ```

   ```
   /**
   	 * Return the cache key for the given view name and the given locale.
   	 * <p>Default is a String consisting of view name and locale suffix.
   	 * Can be overridden in subclasses.
   	 * <p>Needs to respect the locale in general, as a different locale can
   	 * lead to a different view resource.
   	 */
   	protected Object getCacheKey(String viewName, Locale locale) {
   		return viewName + '_' + locale;
   	}
   ```  
   
   ```
   /**
   	 * Create the actual View object.
   	 * <p>The default implementation delegates to {@link #loadView}.
   	 * This can be overridden to resolve certain view names in a special fashion,
   	 * before delegating to the actual {@code loadView} implementation
   	 * provided by the subclass.
   	 * @param viewName the name of the view to retrieve
   	 * @param locale the Locale to retrieve the view for
   	 * @return the View instance, or {@code null} if not found
   	 * (optional, to allow for ViewResolver chaining)
   	 * @throws Exception if the view couldn't be resolved
   	 * @see #loadView
   	 */
   	protected View createView(String viewName, Locale locale) throws Exception {
   		return loadView(viewName, locale);
   	}
   
   	/**
   	 * Subclasses must implement this method, building a View object
   	 * for the specified view. The returned View objects will be
   	 * cached by this ViewResolver base class.
   	 * <p>Subclasses are not forced to support internationalization:
   	 * A subclass that does not may simply ignore the locale parameter.
   	 * @param viewName the name of the view to retrieve
   	 * @param locale the Locale to retrieve the view for
   	 * @return the View instance, or {@code null} if not found
   	 * (optional, to allow for ViewResolver chaining)
   	 * @throws Exception if the view couldn't be resolved
   	 * @see #resolveViewName
   	 */
   	protected abstract View loadView(String viewName, Locale locale) throws Exception;  // 留给子类继承  
   ```
   
   
##### 3. UrlBasedViewResolver extends AbstractCachingViewResolver implements Ordered   
   
   ###### 3.1) 属性  
   ```
   /**
   	 * Prefix for special view names that specify a redirect URL (usually
   	 * to a controller after a form has been submitted and processed).
   	 * Such view names will not be resolved in the configured default
   	 * way but rather be treated as special shortcut.
   	 */
   	public static final String REDIRECT_URL_PREFIX = "redirect:";
   
   	/**
   	 * Prefix for special view names that specify a forward URL (usually
   	 * to a controller after a form has been submitted and processed).
   	 * Such view names will not be resolved in the configured default
   	 * way but rather be treated as special shortcut.
   	 */
   	public static final String FORWARD_URL_PREFIX = "forward:";
   
   
   	private Class<?> viewClass;
   
   	private String prefix = "";
   
   	private String suffix = "";
   
   	private String contentType;
   
   	private boolean redirectContextRelative = true;
   
   	private boolean redirectHttp10Compatible = true;
   
   	private String[] redirectHosts;
   
   	private String requestContextAttribute;
   
   	/** Map of static attributes, keyed by attribute name (String) */
   	private final Map<String, Object> staticAttributes = new HashMap<String, Object>();
   
   	private Boolean exposePathVariables;
   
   	private Boolean exposeContextBeansAsAttributes;
   
   	private String[] exposedContextBeanNames;
   
   	private String[] viewNames;
   
   	private int order = Ordered.LOWEST_PRECEDENCE;
   ```


   ###### 3.2) 初始化    
   ```
   @Override
   	protected void initApplicationContext() {
   		super.initApplicationContext();
   		if (getViewClass() == null) {  // 仅仅判断需要设置ViewClass
   			throw new IllegalArgumentException("Property 'viewClass' is required");
   		}
   	}
   ```
   
   ###### 3.3) 重写AbstractCachingViewResolver方法      
   
   ```
   /**
   	 * This implementation returns just the view name,
   	 * as this ViewResolver doesn't support localized resolution.
   	 */
   	@Override
   	protected Object getCacheKey(String viewName, Locale locale) {
   		return viewName;
   	}
   
   	/**
   	 * Overridden to implement check for "redirect:" prefix.
   	 * <p>Not possible in {@code loadView}, since overridden
   	 * {@code loadView} versions in subclasses might rely on the
   	 * superclass always creating instances of the required view class.
   	 * @see #loadView
   	 * @see #requiredViewClass
   	 */
   	@Override
   	protected View createView(String viewName, Locale locale) throws Exception {
   		// If this resolver is not supposed to handle the given view,
   		// return null to pass on to the next resolver in the chain.
   		if (!canHandle(viewName, locale)) {
   			return null;
   		}
   
   		// Check for special "redirect:" prefix.
   		if (viewName.startsWith(REDIRECT_URL_PREFIX)) {  // 创建重定向View
   			String redirectUrl = viewName.substring(REDIRECT_URL_PREFIX.length());
   			RedirectView view = new RedirectView(redirectUrl,
   					isRedirectContextRelative(), isRedirectHttp10Compatible());
   			view.setHosts(getRedirectHosts());
   			return applyLifecycleMethods(REDIRECT_URL_PREFIX, view);
   		}
   
   		// Check for special "forward:" prefix.
   		if (viewName.startsWith(FORWARD_URL_PREFIX)) {
   			String forwardUrl = viewName.substring(FORWARD_URL_PREFIX.length());
   			return new InternalResourceView(forwardUrl);  // 转发内部资源, 可以共享request资源    
   		}
   
   		// Else fall back to superclass implementation: calling loadView.
   		return super.createView(viewName, locale);
   	}
   ```
   
   ###### 3.4) 实现AbstractCachingViewResolver方法
   ```
   /**
   	 * Delegates to {@code buildView} for creating a new instance of the
   	 * specified view class. Applies the following Spring lifecycle methods
   	 * (as supported by the generic Spring bean factory):
   	 * <ul>
   	 * <li>ApplicationContextAware's {@code setApplicationContext}
   	 * <li>InitializingBean's {@code afterPropertiesSet}
   	 * </ul>
   	 * @param viewName the name of the view to retrieve
   	 * @return the View instance
   	 * @throws Exception if the view couldn't be resolved
   	 * @see #buildView(String)
   	 * @see org.springframework.context.ApplicationContextAware#setApplicationContext
   	 * @see org.springframework.beans.factory.InitializingBean#afterPropertiesSet
   	 */
   	@Override
   	protected View loadView(String viewName, Locale locale) throws Exception {
   		AbstractUrlBasedView view = buildView(viewName);
   		View result = applyLifecycleMethods(viewName, view);
   		return (view.checkResource(locale) ? result : null);
   	}
   ```
   
   ```
   /**
   	 * Creates a new View instance of the specified view class and configures it.
   	 * Does <i>not</i> perform any lookup for pre-defined View instances.
   	 * <p>Spring lifecycle methods as defined by the bean container do not have to
   	 * be called here; those will be applied by the {@code loadView} method
   	 * after this method returns.
   	 * <p>Subclasses will typically call {@code super.buildView(viewName)}
   	 * first, before setting further properties themselves. {@code loadView}
   	 * will then apply Spring lifecycle methods at the end of this process.
   	 * @param viewName the name of the view to build
   	 * @return the View instance
   	 * @throws Exception if the view couldn't be resolved
   	 * @see #loadView(String, java.util.Locale)
   	 */
   	protected AbstractUrlBasedView buildView(String viewName) throws Exception {
   		AbstractUrlBasedView view = (AbstractUrlBasedView) BeanUtils.instantiateClass(getViewClass());
   		view.setUrl(getPrefix() + viewName + getSuffix());
   
   		String contentType = getContentType();
   		if (contentType != null) {
   			view.setContentType(contentType);
   		}
   
   		view.setRequestContextAttribute(getRequestContextAttribute());
   		view.setAttributesMap(getAttributesMap());
   
   		Boolean exposePathVariables = getExposePathVariables();
   		if (exposePathVariables != null) {
   			view.setExposePathVariables(exposePathVariables);
   		}
   		Boolean exposeContextBeansAsAttributes = getExposeContextBeansAsAttributes();
   		if (exposeContextBeansAsAttributes != null) {
   			view.setExposeContextBeansAsAttributes(exposeContextBeansAsAttributes);
   		}
   		String[] exposedContextBeanNames = getExposedContextBeanNames();
   		if (exposedContextBeanNames != null) {
   			view.setExposedContextBeanNames(exposedContextBeanNames);
   		}
   
   		return view;
   	}
   ```   
   
   ```
   private View applyLifecycleMethods(String viewName, AbstractView view) {
   		return (View) getApplicationContext().getAutowireCapableBeanFactory().initializeBean(view, viewName);
   	}
   ```

##### 4. FreeMarkerViewResolver extends AbstractTemplateViewResolver     
   ```
   /**
   	 * Sets the default {@link #setViewClass view class} to {@link #requiredViewClass}:
   	 * by default {@link FreeMarkerView}.
   	 */
   	public FreeMarkerViewResolver() {
   		setViewClass(requiredViewClass());
   	}
   
   	/**
   	 * A convenience constructor that allows for specifying {@link #setPrefix prefix}
   	 * and {@link #setSuffix suffix} as constructor arguments.
   	 * @param prefix the prefix that gets prepended to view names when building a URL
   	 * @param suffix the suffix that gets appended to view names when building a URL
   	 * @since 4.3
   	 */
   	public FreeMarkerViewResolver(String prefix, String suffix) {
   		this();
   		setPrefix(prefix);
   		setSuffix(suffix);
   	}
   
   
   // 主要是设置ViewClass
   	/**
   	 * Requires {@link FreeMarkerView}.
   	 */
   	@Override
   	protected Class<?> requiredViewClass() {
   		return FreeMarkerView.class;
   	}

   ```

   总结, ViewResolver主要是用于查找构建View对象实例, 附加额外的公共的操作,  如缓存, 基于url的查找视图, 
   对于视图的渲染操作, 则委托于View对象   


##### 5. View视图渲染  
   对于视图的渲染操作, 则委托于View对象                                   
   
   View              
       ^         
       |   String getContentType();             
       |   void render(Map<String, ?> model, HttpServletRequest request, 
       |                   HttpServletResponse response) throws Exception;       
       |                                                                                                                                                                                  
   AbstractView             
       ^                
      | |   实现getContentType(), render()         
      | |   
      | | ======================================================== InternalResourceView, RedirectView           
   AbstractUrlBasedView      
       ^       
      | |   afterPropertiesSet() 判断url是必设置属性      
      | |     
   AbstractTemplateView       
       ^     
      | |   重写AbstractView中的renderMergedOutputModel()方法     
      | |              
   FreeMarkerView      
      | |     
      | |   重写AbstractTemplateView中的renderMergedTemplateModel()方法   
      | |     
      


##### 6. AbstractView     

   ###### 6.1) 属性  
   ```
   private String contentType = DEFAULT_CONTENT_TYPE;
   
   	private String requestContextAttribute;
   
   	private final Map<String, Object> staticAttributes = new LinkedHashMap<String, Object>();
   
   	private boolean exposePathVariables = true;
   
   	private boolean exposeContextBeansAsAttributes = false;
   
   	private Set<String> exposedContextBeanNames;
   
   	private String beanName;
   ```

   ###### 6.2) 实现接口方法  
   
   ```
   /**
   	 * Prepares the view given the specified model, merging it with static
   	 * attributes and a RequestContext attribute, if necessary.
   	 * Delegates to renderMergedOutputModel for the actual rendering.
   	 * @see #renderMergedOutputModel
   	 */
   	@Override
   	public void render(Map<String, ?> model, HttpServletRequest request, HttpServletResponse response) throws Exception {
   		if (logger.isTraceEnabled()) {
   			logger.trace("Rendering view with name '" + this.beanName + "' with model " + model +
   				" and static attributes " + this.staticAttributes);
   		}
   
   		Map<String, Object> mergedModel = createMergedOutputModel(model, request, response);
   		prepareResponse(request, response);
   		renderMergedOutputModel(mergedModel, getRequestToExpose(request), response);
   	}
   ```     
   ```
   /**
   	 * Creates a combined output Map (never {@code null}) that includes dynamic values and static attributes.
   	 * Dynamic values take precedence over static attributes.
   	 */
   	protected Map<String, Object> createMergedOutputModel(Map<String, ?> model, HttpServletRequest request,
   			HttpServletResponse response) {
   
   		@SuppressWarnings("unchecked")
   		Map<String, Object> pathVars = (this.exposePathVariables ?
   				(Map<String, Object>) request.getAttribute(View.PATH_VARIABLES) : null);
   
   		// Consolidate static and dynamic model attributes.
   		int size = this.staticAttributes.size();
   		size += (model != null ? model.size() : 0);
   		size += (pathVars != null ? pathVars.size() : 0);
   
   		Map<String, Object> mergedModel = new LinkedHashMap<String, Object>(size);
   		mergedModel.putAll(this.staticAttributes);
   		if (pathVars != null) {
   			mergedModel.putAll(pathVars);
   		}
   		if (model != null) {
   			mergedModel.putAll(model);
   		}
   
   		// Expose RequestContext?
   		if (this.requestContextAttribute != null) {
   			mergedModel.put(this.requestContextAttribute, createRequestContext(request, response, mergedModel));
   		}
   
   		return mergedModel;
   	}
   
   /**
   	 * Subclasses must implement this method to actually render the view.
   	 * <p>The first step will be preparing the request: In the JSP case,
   	 * this would mean setting model objects as request attributes.
   	 * The second step will be the actual rendering of the view,
   	 * for example including the JSP via a RequestDispatcher.
   	 * @param model combined output Map (never {@code null}),
   	 * with dynamic values taking precedence over static attributes
   	 * @param request current HTTP request
   	 * @param response current HTTP response
   	 * @throws Exception if rendering failed
   	 */
   	protected abstract void renderMergedOutputModel(
   			Map<String, Object> model, HttpServletRequest request, HttpServletResponse response) throws Exception;
   ```
    
    
##### 7. AbstractUrlBasedView     

   ###### 7.1) 属性
   ```
   private String url;
   ```
   
   ###### 7.2) 检查url
   ```
   @Override
   	public void afterPropertiesSet() throws Exception {
   		if (isUrlRequired() && getUrl() == null) {
   			throw new IllegalArgumentException("Property 'url' is required");
   		}
   	}
   ```
    
##### 8. AbstractTemplateView extends AbstractUrlBasedView  

   ###### 8.1) 属性  
   ```
   private boolean exposeRequestAttributes = false;
   
   	private boolean allowRequestOverride = false;
   
   	private boolean exposeSessionAttributes = false;
   
   	private boolean allowSessionOverride = false;
   
   	private boolean exposeSpringMacroHelpers = true;
   ```         
    
   ###### 8.2) 实现父类方法
   ```
   @Override
   	protected final void renderMergedOutputModel(
   			Map<String, Object> model, HttpServletRequest request, HttpServletResponse response) throws Exception {
   
   		if (this.exposeRequestAttributes) {
   			for (Enumeration<String> en = request.getAttributeNames(); en.hasMoreElements();) {
   				String attribute = en.nextElement();
   				if (model.containsKey(attribute) && !this.allowRequestOverride) {
   					throw new ServletException("Cannot expose request attribute '" + attribute +
   						"' because of an existing model object of the same name");
   				}
   				Object attributeValue = request.getAttribute(attribute);
   				if (logger.isDebugEnabled()) {
   					logger.debug("Exposing request attribute '" + attribute +
   							"' with value [" + attributeValue + "] to model");
   				}
   				model.put(attribute, attributeValue);  // 从request设置属性到model
   			}
   		}
   
   		if (this.exposeSessionAttributes) {
   			HttpSession session = request.getSession(false);
   			if (session != null) {
   				for (Enumeration<String> en = session.getAttributeNames(); en.hasMoreElements();) {
   					String attribute = en.nextElement();
   					if (model.containsKey(attribute) && !this.allowSessionOverride) {
   						throw new ServletException("Cannot expose session attribute '" + attribute +
   							"' because of an existing model object of the same name");
   					}
   					Object attributeValue = session.getAttribute(attribute);
   					if (logger.isDebugEnabled()) {
   						logger.debug("Exposing session attribute '" + attribute +
   								"' with value [" + attributeValue + "] to model");
   					}
   					model.put(attribute, attributeValue); // 从session设置属性到model
   				}
   			}
   		}
   
   		if (this.exposeSpringMacroHelpers) {
   			if (model.containsKey(SPRING_MACRO_REQUEST_CONTEXT_ATTRIBUTE)) {
   				throw new ServletException(
   						"Cannot expose bind macro helper '" + SPRING_MACRO_REQUEST_CONTEXT_ATTRIBUTE +
   						"' because of an existing model object of the same name");
   			}
   			// Expose RequestContext instance for Spring macros.
   			model.put(SPRING_MACRO_REQUEST_CONTEXT_ATTRIBUTE,
   					new RequestContext(request, response, getServletContext(), model));
   		}
   
   		applyContentType(response);
   
   		renderMergedTemplateModel(model, request, response);
   	}
   ``` 
   
   
   ```
   /**
   	 * Subclasses must implement this method to actually render the view.
   	 * @param model combined output Map, with request attributes and
   	 * session attributes merged into it if required
   	 * @param request current HTTP request
   	 * @param response current HTTP response
   	 * @throws Exception if rendering failed
   	 */
   	protected abstract void renderMergedTemplateModel(
   			Map<String, Object> model, HttpServletRequest request, HttpServletResponse response) throws Exception;
   ```
    
 
 
 
 ##### 9. FreeMarkerView extends AbstractTemplateView
   
   ###### 9.1) 属性    
 
   ```
   private String encoding;
   
   	private Configuration configuration;
   
   	private TaglibFactory taglibFactory;
   
   	private ServletContextHashModel servletContextHashModel;
   ```
   
  ###### 9.2) 初始化  
  ```
  /**
  	 * Invoked on startup. Looks for a single FreeMarkerConfig bean to
  	 * find the relevant Configuration for this factory.
  	 * <p>Checks that the template for the default Locale can be found:
  	 * FreeMarker will check non-Locale-specific templates if a
  	 * locale-specific one is not found.
  	 * @see freemarker.cache.TemplateCache#getTemplate
  	 */
  	@Override
  	protected void initServletContext(ServletContext servletContext) throws BeansException {
  		if (getConfiguration() != null) {
  			this.taglibFactory = new TaglibFactory(servletContext);
  		}
  		else {
  			FreeMarkerConfig config = autodetectConfiguration();
  			setConfiguration(config.getConfiguration());
  			this.taglibFactory = config.getTaglibFactory();
  		}
  
  		GenericServlet servlet = new GenericServletAdapter();
  		try {
  			servlet.init(new DelegatingServletConfig());
  		}
  		catch (ServletException ex) {
  			throw new BeanInitializationException("Initialization of GenericServlet adapter failed", ex);
  		}
  		this.servletContextHashModel = new ServletContextHashModel(servlet, getObjectWrapper());
  	}
  ``` 
 
  ###### 9.3) 重写AbstractUrlBasedView方法   
  ```
  /**
  	 * Check that the FreeMarker template used for this view exists and is valid.
  	 * <p>Can be overridden to customize the behavior, for example in case of
  	 * multiple templates to be rendered into a single view.
  	 */
  	@Override
  	public boolean checkResource(Locale locale) throws Exception {
  		String url = getUrl();
  		try {
  			// Check that we can get the template, even if we might subsequently get it again.
  			getTemplate(url, locale);
  			return true;
  		}
  		catch (FileNotFoundException ex) {
  			if (logger.isDebugEnabled()) {
  				logger.debug("No FreeMarker view found for URL: " + url);
  			}
  			return false;
  		}
  		catch (ParseException ex) {
  			throw new ApplicationContextException(
  					"Failed to parse FreeMarker template for URL [" + url + "]", ex);
  		}
  		catch (IOException ex) {
  			throw new ApplicationContextException(
  					"Could not load FreeMarker template for URL [" + url + "]", ex);
  		}
  	}
  ```     
    
  ###### 9.4) 重写AbstractTemplateView方法  
  
  ```
  /**
  	 * Process the model map by merging it with the FreeMarker template.
  	 * Output is directed to the servlet response.
  	 * <p>This method can be overridden if custom behavior is needed.
  	 */
  	@Override
  	protected void renderMergedTemplateModel(
  			Map<String, Object> model, HttpServletRequest request, HttpServletResponse response) throws Exception {
  
  		exposeHelpers(model, request);
  		doRender(model, request, response);
  	}
  ```
  
  ```
  /**
  	 * Render the FreeMarker view to the given response, using the given model
  	 * map which contains the complete template model to use.
  	 * <p>The default implementation renders the template specified by the "url"
  	 * bean property, retrieved via {@code getTemplate}. It delegates to the
  	 * {@code processTemplate} method to merge the template instance with
  	 * the given template model.
  	 * <p>Adds the standard Freemarker hash models to the model: request parameters,
  	 * request, session and application (ServletContext), as well as the JSP tag
  	 * library hash model.
  	 * <p>Can be overridden to customize the behavior, for example to render
  	 * multiple templates into a single view.
  	 * @param model the model to use for rendering
  	 * @param request current HTTP request
  	 * @param response current servlet response
  	 * @throws IOException if the template file could not be retrieved
  	 * @throws Exception if rendering failed
  	 * @see #setUrl
  	 * @see org.springframework.web.servlet.support.RequestContextUtils#getLocale
  	 * @see #getTemplate(java.util.Locale)
  	 * @see #processTemplate
  	 * @see freemarker.ext.servlet.FreemarkerServlet
  	 */
  	protected void doRender(Map<String, Object> model, HttpServletRequest request, HttpServletResponse response) throws Exception {
  		// Expose model to JSP tags (as request attributes).
  		exposeModelAsRequestAttributes(model, request);
  		// Expose all standard FreeMarker hash models.
  		SimpleHash fmModel = buildTemplateModel(model, request, response);
  
  		if (logger.isDebugEnabled()) {
  			logger.debug("Rendering FreeMarker template [" + getUrl() + "] in FreeMarkerView '" + getBeanName() + "'");
  		}
  		// Grab the locale-specific version of the template.
  		Locale locale = RequestContextUtils.getLocale(request);
  		processTemplate(getTemplate(locale), fmModel, response);
  	}
  ```  
                    