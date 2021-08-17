---
layout: post
title:  "深入剖析 tomcat nio +linux epoll 模型"
date:   2020-05-10 00:06:05
categories: tomcat
tags: tomcat 源码
---

##### springboot , tomcat nio +linux epoll 模型
-   1.springboot 默认启动tomcat时候 会从 TomcatServletWebServerFactory 获取默认协议而是 org.apache.coyote.http11.Http11NioProtocol
相关代码如下:


		public class TomcatServletWebServerFactory extends AbstractServletWebServerFactory
				implements ConfigurableTomcatWebServerFactory, ResourceLoaderAware {
	
		private static final Charset DEFAULT_CHARSET = StandardCharsets.UTF_8;
	
		private static final Set<Class<?>> NO_CLASSES = Collections.emptySet();
	
		/**
		 * The class name of default protocol used.
		 * @describle springboot 默认使用的是 org.apache.coyote.http11.Http11NioProtocol协议 ---> NioEndPoint
		 */
		public static final String DEFAULT_PROTOCOL = "org.apache.coyote.http11.Http11NioProtocol";
	
		@Override
		public WebServer getWebServer(ServletContextInitializer... initializers) {
			if (this.disableMBeanRegistry) {
				Registry.disableRegistry();
			}
			Tomcat tomcat = new Tomcat();
			File baseDir = (this.baseDirectory != null) ? this.baseDirectory : createTempDir("tomcat");
			tomcat.setBaseDir(baseDir.getAbsolutePath());
			Connector connector = new Connector(this.protocol);
			connector.setThrowOnFailure(true);
			tomcat.getService().addConnector(connector);
			customizeConnector(connector);
			tomcat.setConnector(connector);
			tomcat.getHost().setAutoDeploy(false);
			configureEngine(tomcat.getEngine());
			for (Connector additionalConnector : this.additionalTomcatConnectors) {
				tomcat.getService().addConnector(additionalConnector);
			}
			prepareContext(tomcat.getHost(), initializers);
			return getTomcatWebServer(tomcat);
		}
	}


-   2.Connector创建的时候需要构造参数，在connector 实例化的过程中会根据协议确定当前对象采取io模型， 构造方法如下

		/**
		     * @describle 实例化connector 对象
		     * @param protocol
		     */
		    public Connector(String protocol) {
		        boolean aprConnector = AprLifecycleListener.isAprAvailable() &&
		                AprLifecycleListener.getUseAprConnector();
		
	        /**
	         * 默认是 org.apache.coyote.http11.Http11NioProtocol
	         */
	        if ("HTTP/1.1".equals(protocol) || protocol == null) {
	            if (aprConnector) {
	                protocolHandlerClassName = "org.apache.coyote.http11.Http11AprProtocol";
	            } else {
	                protocolHandlerClassName = "org.apache.coyote.http11.Http11NioProtocol";
	            }
	        } else if ("AJP/1.3".equals(protocol)) {
	            if (aprConnector) {
	                protocolHandlerClassName = "org.apache.coyote.ajp.AjpAprProtocol";
	            } else {
	                protocolHandlerClassName = "org.apache.coyote.ajp.AjpNioProtocol";
	            }
	        } else {
	            protocolHandlerClassName = protocol;
	        }
	
	        /**
	         * @describle 实例化协议处理器  默认是NioEndpoint
	         */
	        // Instantiate protocol handler
	        ProtocolHandler p = null;
	        try {
	            Class<?> clazz = Class.forName(protocolHandlerClassName);
	            p = (ProtocolHandler) clazz.getConstructor().newInstance();
	        } catch (Exception e) {
	            log.error(sm.getString(
	                    "coyoteConnector.protocolHandlerInstantiationFailed"), e);
	        } finally {
	            this.protocolHandler = p;
	        }
	
	        // Default for Connector depends on this system property
	        setThrowOnFailure(Boolean.getBoolean("org.apache.catalina.startup.EXIT_ON_INIT_FAILURE"));
	    }


##### jdk nio + linux下jdk nio模型

-   1.jdk在linux下面nio模型使用的是 EPollSelectorImpl 在windows 下面使用的是WindowsSelectorImpl

-   a.EPollSelectorImpl默认选择配置，如果系统没有配置 java.nio.channels.spi.SelectorProvider，那么jdk会默认调用provider = sun.nio.ch.DefaultSelectorProvider.create();而DefaultSelectorProvider在不同的操作系统系统会获取不同的Provider代码如下

		 public static SelectorProvider provider() {
		        synchronized (lock) {
		            if (provider != null)
		                return provider;
		            return AccessController.doPrivileged(
		                new PrivilegedAction<SelectorProvider>() {
		                    public SelectorProvider run() {
		                            if (loadProviderFromProperty())
		                                return provider;
		                            if (loadProviderAsService())
		                                return provider;
		                            provider = sun.nio.ch.DefaultSelectorProvider.create();
		                            return provider;
		                        }
		                    });
		        }
		    }
		
		
		
		/**
		 * Creates this platform's default SelectorProvider
		 */
		
		public class DefaultSelectorProvider {
	
	    /**
	     * Prevent instantiation.
	     */
	    private DefaultSelectorProvider() { }
	
	    @SuppressWarnings("unchecked")
	    private static SelectorProvider createProvider(String cn) {
	        Class<SelectorProvider> c;
	        try {
	            c = (Class<SelectorProvider>)Class.forName(cn);
	        } catch (ClassNotFoundException x) {
	            throw new AssertionError(x);
	        }
	        try {
	            return c.newInstance();
	        } catch (IllegalAccessException | InstantiationException x) {
	            throw new AssertionError(x);
	        }
	
	    }
	
	    /**
	     * Returns the default SelectorProvider.
	     */
	    public static SelectorProvider create() {
	        String osname = AccessController
	            .doPrivileged(new GetPropertyAction("os.name"));
	        if (osname.equals("SunOS"))
	            return createProvider("sun.nio.ch.DevPollSelectorProvider");
	        if (osname.equals("Linux"))
	            return createProvider("sun.nio.ch.EPollSelectorProvider");
	        return new sun.nio.ch.PollSelectorProvider();
	    }
	
	} 


-   2.tomcat 之nio 事件处理流程,首先看一下时序图

<div align="left">  
	<img src="https://kunge2013.go123.live/images/frame/tomcat/tomcatnio 时序图.png" width="100%"/>
</div>


-   3.代码如下
[相关源码](https://github.com/kunge2013/springwebmvc.git)

