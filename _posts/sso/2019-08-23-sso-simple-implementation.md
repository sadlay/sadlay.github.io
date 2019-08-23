---
layout: post
title: 单点登陆简单实现（完全跨域、单点退出）
categories: Sso
description: 单点登陆基于Session的简单实现，包括完全跨域、单点退出等。
keywords: sso, Java
---


# 单点登陆简单实现（完全跨域、单点退出）

## 单点登陆活动图

![img](http://img.blog.csdn.net/20150313132409301?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbHV4aWFucGluZw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

废话不多说直接上代码咯，什么好处，实现原理相关资料baidu去，基本都一样，就不复制了



## 1 服务端代码

### 1.1 登陆跳转注销代码   

controller层代码

```java
package com.ffcs.sso.controller;  
  
import javax.servlet.http.HttpSession;  
  
import org.apache.commons.lang.StringUtils;  
import org.slf4j.Logger;  
import org.slf4j.LoggerFactory;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.stereotype.Controller;  
import org.springframework.ui.Model;  
import org.springframework.web.bind.annotation.RequestMapping;  
import org.springframework.web.bind.annotation.RequestMethod;  
import org.springframework.web.bind.annotation.RequestParam;  
import org.springframework.web.servlet.ModelAndView;  
  
import com.ffcs.sso.pojo.User;  
import com.ffcs.sso.service.LogoutManagerService;  
import com.ffcs.sso.service.TicketManagerService;  
import com.ffcs.sso.service.UserManagerService;  
import com.ffcs.sso.utils.Constant;  
/** 
 *  
 * @author damon 
 * 
 */  
@Controller  
public class SSOController {  
      
    private final static Logger LOGGER=LoggerFactory.getLogger(SSOController.class);  
   
    @Autowired  
    private TicketManagerService ticketManagerService;  
      
    @Autowired  
    private UserManagerService userManagerService;  
      
    @Autowired  
    private LogoutManagerService logoutManagerService;  
  
    /** 
     * 主页 
     * @return 
     */  
    @RequestMapping(value="manager/index",method=RequestMethod.GET)  
    public String index(){  
          
        return "index";  
    }  
    /** 
     * 登陆首页 
     * @return 
     */  
    @RequestMapping(value="login",method=RequestMethod.GET)  
    public ModelAndView loginIndex(String queryParams,String targetUrl,ModelAndView modelAndView){  
          
        modelAndView.addObject("queryParams", queryParams);  
          
        modelAndView.addObject("targetUrl", targetUrl);  
          
        modelAndView.setViewName("login");  
          
        return modelAndView;  
    }  
      
    @RequestMapping(value="login",method=RequestMethod.POST)  
    public String login(@RequestParam(required = true) String account, @RequestParam(required = true) String password,  
  
            String targetUrl, String queryParams, Model model, HttpSession session) {  
  
        User user = userManagerService.getUserInfo(account, password);  
  
        if (user != null) {  
  
            String homeCookieId = session.getId();  
  
            session.setAttribute(Constant.SSO_IS_LOGIN, true);  
  
            session.setAttribute(Constant.SSO_LOGIG_INFO, account);  
  
            logoutManagerService.saveLoginUserInfo(homeCookieId, user);  
  
            if (StringUtils.isNotBlank(targetUrl)) {  
  
                // 生成targetURL对应的票  
  
                String ticket = ticketManagerService.generateTicket(targetUrl, homeCookieId);  
  
                String params = StringUtils.isNotBlank(queryParams) ? "&" + queryParams : "";  
  
                LOGGER.info("###################  用户账号:[{}]  对应主站cookieId:[{}] 登陆系统 :[{}]", account, homeCookieId, targetUrl);  
  
                return "redirect:" + targetUrl + "?" + Constant.SSO_TICKET + "=" + ticket + params;  
  
            }  
            return "redirect:manager/index";  
  
        } else {  
  
            model.addAttribute("error", "用户名或密码错误！");  
  
            if (StringUtils.isNotBlank(targetUrl)) {  
  
                model.addAttribute("targetUrl", targetUrl);  
            }  
            return "login";  
        }  
    }  
      
    /** 
     * 重定向到子系统并生成票 
     * @param targetUrl 
     * @param queryParams 
     * @param modelAndView 
     * @param session 
     * @return 
     */  
    @RequestMapping(value="redirect",method=RequestMethod.GET)  
    public ModelAndView redirect(@RequestParam(value="targetUrl",required=true)String targetUrl,  
              
            String queryParams,ModelAndView modelAndView,HttpSession session) {  
      
        if(session.getAttribute(Constant.SSO_IS_LOGIN)==null){  
              
            modelAndView.setViewName("redirect:login");  
              
            modelAndView.addObject("targetUrl", targetUrl);  
              
            modelAndView.addObject("queryParams", queryParams);       
          
            //重定向到login方法，带上目标网页地址  
          
        }else{  
              
            String homeCookieId=session.getId();  
              
            String account=(String) session.getAttribute(Constant.SSO_LOGIG_INFO);  
              
            //生成targetURL对应的票  
              
            String ticket=ticketManagerService.generateTicket(targetUrl,homeCookieId);  
                      
            String params=StringUtils.isNotBlank(queryParams)?"&"+queryParams:"";  
              
            modelAndView.setViewName("redirect:" + targetUrl + "?" + Constant.SSO_TICKET + "=" + ticket + params);    
  
            LOGGER.info("############### 用户账号:[{}] 主站cookieId:[{}] 重定向到系统:[{}] 对应ticket:[{}]", account, homeCookieId, targetUrl, ticket);  
        }  
          
        return modelAndView;  
    }  
      
    /** 
     * 单点注销 
     * @param session 
     * @return 
     */  
    @RequestMapping(value="logout",method=RequestMethod.GET)  
    public String logout(HttpSession session){  
          
        if(session.getAttribute(Constant.SSO_IS_LOGIN)!=null){  
          
            String cookieId=session.getId();  
              
            String account=(String) session.getAttribute(Constant.SSO_LOGIG_INFO);  
              
            logoutManagerService.logout(cookieId);  
              
            session.invalidate();     
                  
            LOGGER.info("########### 单点退出用户账号:[{}] 对应主站cookieId为:[{}] ",account,cookieId);  
        }  
        return "redirect:login";  
    }     
      
}  
```

### 1.2 生成票管理接口代码（对外restful webservice发布）

```java
package com.ffcs.sso.service;  
  
import java.util.UUID;  
  
import net.rubyeye.xmemcached.MemcachedClient;  
  
import org.apache.commons.lang.StringUtils;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.beans.factory.annotation.Qualifier;  
import org.springframework.stereotype.Service;  
  
import com.ffcs.sso.exception.SSOException;  
import com.ffcs.sso.pojo.TicketInfo;  
import com.ffcs.sso.pojo.TicketResponseInfo;  
import com.ffcs.sso.pojo.User;  
import com.ffcs.sso.utils.Constant;  
/** 
 *  
 * 票管理 
 *  
 *  
 *  damon 
 * 
 */  
@Service  
public class TicketManagerServiceImpl implements TicketManagerService{    
      
    @Autowired  
    @Qualifier(value="memcachedSSOClient")  
    private MemcachedClient memcachedClient;  
    @Autowired  
    private LogoutManagerService logoutManagerService;  
  
    @Override  
    public String generateTicket(String target,String homeCookieId){  
          
        if(StringUtils.isBlank(target)){  
              
            throw new IllegalArgumentException("target不可以为空！");  
              
        }  
          
        if(StringUtils.isBlank(homeCookieId)){  
              
            throw new IllegalArgumentException("homeCookieId不可以为空！");  
              
        }  
          
        String ticket=UUID.randomUUID().toString();  
      
        try {  
              
            memcachedClient.set(ticket, Constant.MAX_TICKET_INACTIVE_INTERVAL, new TicketInfo(ticket,target,homeCookieId));  
              
            return ticket;  
              
        } catch (Exception e) {  
  
            throw new SSOException(e);  
        }  
    }  
    @Override   
    public TicketResponseInfo validateTicket(String ticket,String target,  
              
            String subCookieId,String subCookieName,String subLogoutPath){  
          
        if(StringUtils.isBlank(ticket)){  
  
            throw new IllegalArgumentException("ticket不可以为空！");  
        }  
          
        if(StringUtils.isBlank(target)){  
  
            throw new IllegalArgumentException("target不可以为空！");  
        }  
          
        if(StringUtils.isBlank(subCookieId)){  
  
            throw new IllegalArgumentException("subCookieId不可以为空！");  
        }  
  
        if(StringUtils.isBlank(subCookieName)){  
  
            throw new IllegalArgumentException("subCookieName不可以为空！");  
        }  
          
        if(StringUtils.isBlank(subLogoutPath)){  
  
            throw new IllegalArgumentException("subLogoutPath不可以为空！");  
        }  
  
        try {  
              
            TicketInfo ticketInfo = memcachedClient.get(ticket);  
              
            if(ticketInfo==null||!target.equals(ticketInfo.getTargetUrl())){  
                  
                //返回空验证不通过  
                  
                return new TicketResponseInfo(false);  
            }  
              
            //删除票保存的临时信息  
              
            memcachedClient.delete(ticket);  
              
            String homeCookieId=ticketInfo.getHomeCookieId();  
              
            //验证后保存登出信息（原本验证和登出信息分开提交，一并提交减少访问次数）  
              
            logoutManagerService.saveSubWebsiteLogouInfo(homeCookieId, subLogoutPath, subCookieId, subCookieName);  
               
            User user= logoutManagerService.getLoginUserInfo(homeCookieId);  
              
            return new TicketResponseInfo(true,user.getAccount(),homeCookieId);  
  
        } catch (Exception e) {  
              
            throw new SSOException(e);  
        }   
      
    }  
  
  
}  
```

### 1.3 单点退出接口（对外restful webservice发布）

```java
package com.ffcs.sso.service;  
  
import java.util.Set;  
  
import net.rubyeye.xmemcached.MemcachedClient;  
  
import org.apache.commons.lang.StringUtils;  
import org.apache.http.HttpResponse;  
import org.apache.http.client.HttpClient;  
import org.apache.http.client.methods.HttpPost;  
import org.slf4j.Logger;  
import org.slf4j.LoggerFactory;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.beans.factory.annotation.Qualifier;  
import org.springframework.stereotype.Service;  
  
import com.ffcs.sso.exception.SSOException;  
import com.ffcs.sso.pojo.ActivationInfo;  
import com.ffcs.sso.pojo.LogoutInfo;  
import com.ffcs.sso.pojo.User;  
import com.ffcs.sso.utils.Constant;  
  
/** 
 * 单点退出 
 *  
 * damon 
 */  
  
  
@Service  
public class LogoutManagerServiceImpl implements LogoutManagerService {  
      
    private static final Logger LOGGER=LoggerFactory.getLogger(LogoutManagerServiceImpl.class);  
      
    private final String LOGOUT_PREFIX="LOGOUT_";  
  
    @Autowired  
    @Qualifier("memcachedSSOClient")  
    private MemcachedClient memcachedClient;  
      
    @Autowired  
    private HttpClient httpClient;  
  
    @Override  
    public boolean saveSubWebsiteLogouInfo(String homeCookieId, String logoutPath,String subCookieId,String subCookieName) {  
          
        if(StringUtils.isBlank(homeCookieId)){  
              
            throw new IllegalArgumentException("homeCookieId 不可以为空！");  
        }  
        if(StringUtils.isBlank(logoutPath)){  
          
            throw new IllegalArgumentException("logoutPath 不可以为空！");  
        }  
        if(StringUtils.isBlank(subCookieId)){  
              
            throw new IllegalArgumentException("subWebSite 不可以为空！");  
        }  
        ActivationInfo info=null;  
          
        Set<LogoutInfo> logoutInfos=null;  
          
        try {  
              
            info=memcachedClient.get(LOGOUT_PREFIX + homeCookieId);  
              
            if(info==null){  
                  
                info=new ActivationInfo();  
      
            }  
            logoutInfos=info.getLogoutInfo();  
              
            logoutInfos.add(new LogoutInfo(logoutPath,subCookieId,subCookieName));  
              
            info.setLogoutInfo(logoutInfos);  
              
            memcachedClient.set(LOGOUT_PREFIX + homeCookieId, Constant.MAX_USERINFO_INACTIVE_INTERVAL, info);  
                  
            LOGGER.debug("############### 保存子站登出信息 ,子站登出地址 url:{}, 子站cookie:{}, 主站cookie:{}", logoutPath, subCookieId, homeCookieId);  
              
            return true;  
          
        } catch (Exception e) {  
              
            throw new SSOException(e);  
        }   
  
    }  
  
    @Override  
    public void logout(String homeCookieId) {  
          
        if(StringUtils.isBlank(homeCookieId)){  
              
            throw new IllegalArgumentException("cookieId 不可以为空！");  
        }  
           
        ActivationInfo info=null;  
          
        Set<LogoutInfo> logoutInfos=null;  
          
        try {  
              
            info = memcachedClient.get(LOGOUT_PREFIX+homeCookieId);  
              
            memcachedClient.delete(LOGOUT_PREFIX+homeCookieId);  
              
            if(info==null|| (logoutInfos=info.getLogoutInfo())==null){  
                  
                LOGGER.debug("##############  用户 cookieId:[{}] 未登陆任何系统   ！",homeCookieId);  
  
                return;  
            }     
        } catch (Exception e) {  
              
            LOGGER.error("###############   Memcached获取单点登出信息失败", e);  
              
            return;  
        }   
          
        for (LogoutInfo logoutInfo:logoutInfos) {  
  
            HttpPost post=null;  
              
            try {  
  
                post = new HttpPost(logoutInfo.getLogoutPath());  
                  
                post.setHeader("charset", "UTF-8");  
                  
                post.setHeader("Connection", "close");    
                  
                //添加cookie模拟回调时候子系统能找到session  
                  
                post.setHeader("Cookie",logoutInfo.getSubCookieName()+"="+logoutInfo.getSubCookieId());  
  
                HttpResponse response = httpClient.execute(post);  
  
                LOGGER.debug("########## 登出子站  :[{}] 主站cookie:[{}] 子站cookie:[{}]  登出返回状态码:[{}]  ",  
                          
                        logoutInfo.getLogoutPath(), homeCookieId, logoutInfo.getSubCookieId(),  response.getStatusLine().getStatusCode());  
                  
            } catch (Exception e) {  
                  
                LOGGER.error("########## 注销子系统失败,子系统信息:{}",logoutInfo.toString(),e);  
              
            }  finally {  
  
                if(post!=null){  
                      
                    post.releaseConnection();                 
                }  
            }  
        }  
    }  
    @Override  
    public void saveLoginUserInfo(String homeCookieId,User userInfo){  
          
        try {  
              
            ActivationInfo info=memcachedClient.get(LOGOUT_PREFIX+homeCookieId);  
              
            if(info==null){  
                  
                info=new ActivationInfo();  
            }  
              
            info.setUserInfo(userInfo);  
              
            memcachedClient.set(LOGOUT_PREFIX+homeCookieId, Constant.MAX_USERINFO_INACTIVE_INTERVAL, info);  
              
        } catch (Exception e) {  
              
            throw new SSOException("#############  保存用户登陆信息失败 ",e);  
        }  
          
          
    }  
    @Override  
    public User getLoginUserInfo(String homeCookieId){  
          
        ActivationInfo info = null;  
          
        try {  
            info=memcachedClient.get(LOGOUT_PREFIX+homeCookieId);  
              
            if(info!=null){  
                  
                return info.getUserInfo();            
            }  
              
            throw new RuntimeException("找不到该cookie:["+homeCookieId+"]的用户信息");  
              
        } catch (Exception e) {  
              
            throw new SSOException("############# 获取登陆用户信息失败 ####",e);  
        }   
          
    }  
  
    @Override  
    public boolean updateUserInfoTimeout(String homeCookieId) {  
          
        if(StringUtils.isBlank(homeCookieId)){  
              
            throw new IllegalArgumentException("homeCookieId 不可以为空！");  
        }  
          
        try {  
              
            return memcachedClient.touch(LOGOUT_PREFIX+homeCookieId, Constant.MAX_USERINFO_INACTIVE_INTERVAL);  
          
        } catch (Exception e) {  
              
            LOGGER.error("############ 定时更新主站登陆用户信息失败 ",e);  
        }   
        return false;  
  
    }  
      
}  
```

## 2 客户端代码

### 2.1 过滤器代码

```java
package com.ffcs.sso.filter;  
  
import java.io.IOException;  
import java.net.URLEncoder;  
import java.util.Iterator;  
import java.util.Map;  
import java.util.Map.Entry;  
  
import javax.servlet.Filter;  
import javax.servlet.FilterChain;  
import javax.servlet.FilterConfig;  
import javax.servlet.ServletException;  
import javax.servlet.ServletRequest;  
import javax.servlet.ServletResponse;  
import javax.servlet.http.HttpServletRequest;  
import javax.servlet.http.HttpServletResponse;  
import javax.servlet.http.HttpSession;  
  
import org.apache.commons.lang.StringUtils;  
import org.apache.http.client.methods.HttpGet;  
import org.apache.http.client.methods.HttpPut;  
import org.slf4j.Logger;  
import org.slf4j.LoggerFactory;  
  
import com.ffcs.sso.exception.HttpRequestException;  
import com.ffcs.sso.pojo.TicketResponseInfo;  
import com.ffcs.sso.utils.Constant;  
import com.ffcs.sso.utils.HttpClientUtils;  
import com.ffcs.sso.utils.JsonUtil;  
  
public class SSOFilter implements Filter {  
      
    private final static Logger LOGGER=LoggerFactory.getLogger(SSOFilter.class);  
  
    private String ticketValidateURL;  
      
    private String redirectLoginURL;  
      
    private String updateUserInfoTimeOutURL;  
  
    @Override  
    public void destroy() {  
          
    }  
  
    @Override  
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {  
          
        HttpServletRequest request = (HttpServletRequest) req;  
          
        HttpServletResponse response = (HttpServletResponse) res;  
          
        HttpSession session=request.getSession();  
          
        if(session.getAttribute(Constant.SSO_IS_LOGIN)==null){  
          
            String targetUrl = request.getRequestURL().toString();  
  
            String ticket = request.getParameter(Constant.SSO_TICKET);        
              
            //子站单点退出路径  
              
            String subLogoutPath = request.getScheme()+"://"+request.getServerName()+":"  
                      
                        +request.getServerPort()+ request.getContextPath() +"/"+Constant.SSO_LOGOUT_SUFFIX;  
                   
            TicketResponseInfo responseInfo=new TicketResponseInfo(false);  
              
            if(StringUtils.isNotBlank(ticket)){  
                  
                //校验票如果校验通过同时保存子站的登出信息(减少提交次数，原本验证成功后在提交登出信息),并返回主站的cookieId和主站登陆的账号  
                  
                responseInfo=validateTicketInfo(ticket,targetUrl,Constant.SSO_SUB_COOKIE_NAME,session.getId(),subLogoutPath);  
            }  
  
            if(!responseInfo.isSuccess()){  
                  
                String queryParams =this.getQureyParams(request);  
  
                String params=StringUtils.isNotBlank(queryParams)?"&queryParams="+URLEncoder.encode(queryParams,"UTF-8"):"";  
                  
                //重定向到主站判断系统是否登陆过  
                  
                response.sendRedirect(redirectLoginURL + "?targetUrl=" + targetUrl + params);  
                      
                LOGGER.debug("############## 重定向到主系统：" + redirectLoginURL + "?targetUrl=" + targetUrl + params);                      
                  
                return;  
            }  
              
            session.setAttribute(Constant.SSO_IS_LOGIN, true);  
              
            session.setAttribute(Constant.SSO_ACCOUNT, responseInfo.getAccount());  
              
            session.setAttribute(Constant.SSO_HOME_COOKIEID, responseInfo.getHomeCookieId());     
        }  
  
        Long updateInterval = (System.currentTimeMillis() - session.getLastAccessedTime()) / 1000;  
          
        if( updateInterval > Constant.SSO_HOME_USERINFO_UPDATE_TIMEOUT){  
              
            //更新主站用户信息超时时间  
              
            updateUserInfoTimeOut((String)session.getAttribute(Constant.SSO_HOME_COOKIEID));  
                  
            LOGGER.debug("############## 更新主站用户信息超时时间,间隔{}秒   ########",Constant.SSO_HOME_USERINFO_UPDATE_TIMEOUT);                   
      
        }  
          
        chain.doFilter(request, response);  
                  
        return;  
    }  
  
    private String getQureyParams(HttpServletRequest request) {  
          
        StringBuilder queryParams=new StringBuilder();  
          
        Map<String,String[]> map=request.getParameterMap();  
          
        Iterator<Entry<String, String[]>>  params=map.entrySet().iterator();  
          
        boolean tag=false;  
          
        while (params.hasNext()) {  
              
            Map.Entry<String,String[]> entry = params.next();  
              
            if(!entry.getKey().equals(Constant.SSO_TICKET)){  
                  
                String[] values=entry.getValue();  
                  
                for(String value:values){  
  
                    if(tag){  
                          
                        queryParams.append("&");  
                    }  
                    queryParams.append(entry.getKey()).append("=").append(value);                         
                }  
                tag=true;  
            }  
        }  
        return queryParams.toString();  
    }  
      
    private void updateUserInfoTimeOut(String homeCookieId){  
      
        HttpPut put=new HttpPut(updateUserInfoTimeOutURL);  
          
        //添加cookie主站那边能够找到session  
          
        put.setHeader("Cookie",Constant.SSO_HOME_COOKIE_NAME+"="+homeCookieId);  
          
        put.setHeader("accept", "text/plain; charset=UTF-8");  
          
        try {  
              
            HttpClientUtils.getResponse(put);  
              
        } catch (HttpRequestException e) {  
              
            LOGGER.error("###################### 定时更新主站用户信息失败 ",e);  
        }  
    }  
  
    /** 
     * 根据ticket验证是否正确，验证通过返回主站的cookieId 
     *  
     * @param ticket 
     *  
     * @param target 
     *  
     * @return  返回验证空失败  成功cookieId 
     */  
    private TicketResponseInfo validateTicketInfo(String ticket, String target,String subCookieName,  
              
            String subCookieId, String subLogoutPath) {  
          
        HttpGet httpget = new HttpGet(ticketValidateURL + "/"+ ticket + "?target=" + target  
                  
                +"&subCookieId="+subCookieId+"&subCookieName="+subCookieName+"&subLogoutPath="+subLogoutPath);  
          
        httpget.setHeader("accept", "application/json; charset=UTF-8");  
          
        String result;  
          
        try {  
              
            result = HttpClientUtils.getResponse(httpget);  
              
            return JsonUtil.fromJson(result, TicketResponseInfo.class);  
              
        } catch (HttpRequestException e) {  
              
            throw new RuntimeException("############### 子站验证ticket失败  ",e);  
        }     
          
          
    }  
  
    @Override  
    public void init(FilterConfig filterConfig) {  
          
        if (filterConfig != null) {  
              
            LOGGER.info("#######   SSOFilter:Initializing filter");  
        }  
          
        this.ticketValidateURL = filterConfig.getInitParameter("ticketValidateURL");  
          
        this.redirectLoginURL=filterConfig.getInitParameter("redirectLoginURL");  
  
        this.updateUserInfoTimeOutURL=filterConfig.getInitParameter("updateUserInfoTimeOutURL");  
    }  
      
}  
```

### 2.2 登出过滤器代码

比较简单，服务端通过httpclient回调这个路口进行退出，服务端必须把客户端登陆的cookie返回过来，不然找不到session

```java
package com.ffcs.sso.filter;  
  
import java.io.IOException;  
  
import javax.servlet.Filter;  
import javax.servlet.FilterChain;  
import javax.servlet.FilterConfig;  
import javax.servlet.ServletException;  
import javax.servlet.ServletRequest;  
import javax.servlet.ServletResponse;  
import javax.servlet.http.HttpServletRequest;  
  
public class LogoutFilter implements Filter{  
      
    @Override  
    public void destroy() {  
          
          
    }  
  
    @Override  
    public void doFilter(ServletRequest req, ServletResponse res,FilterChain chain) throws IOException, ServletException {  
          
        HttpServletRequest request = (HttpServletRequest) req;  
          
        request.getSession().invalidate();  
          
        return;  
          
    }  
  
    @Override  
    public void init(FilterConfig filterConfig) {  
  
    }  
      
}  
```

### 2.3 客户端web.xml配置

```xml
<?xml version="1.0" encoding="UTF-8"?>  
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
    xmlns="http://java.sun.com/xml/ns/javaee" xmlns:web="http://java.sun.com/xml/ns/javaee"  
    xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"  
    version="3.0">  
    <filter>  
        <filter-name>DisableUrlSessionFilter</filter-name>  
        <filter-class>com.ffcs.sso.filter.DisableUrlSessionFilter</filter-class>  
    </filter>  
    <filter-mapping>  
        <filter-name>DisableUrlSessionFilter</filter-name>  
        <url-pattern>/*</url-pattern>  
    </filter-mapping>  
  
    <filter>  
        <filter-name>sessionFilter</filter-name>  
        <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>  
    </filter>  
    <filter-mapping>  
        <filter-name>sessionFilter</filter-name>  
        <url-pattern>/*</url-pattern>  
    </filter-mapping>  
  
    <filter>  
        <filter-name>LogoutFilter</filter-name>  
        <filter-class>com.ffcs.sso.filter.LogoutFilter</filter-class>  
    </filter>  
    <filter-mapping>  
        <filter-name>LogoutFilter</filter-name>  
        <url-pattern>/SSO_LOGOUT</url-pattern>  
    </filter-mapping>  
    <listener>  
        <description>spring监听器</description>  
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>  
    </listener>  
    <listener>  
        <listener-class>org.springframework.web.util.IntrospectorCleanupListener</listener-class>  
    </listener>  
    <context-param>  
        <param-name>contextConfigLocation</param-name>  
        <param-value>classpath:spring.xml</param-value>  
    </context-param>  
  
    <filter>  
        <description>字符集过滤器</description>  
        <filter-name>encodingFilter</filter-name>  
        <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>  
        <init-param>  
            <description>字符集编码</description>  
            <param-name>encoding</param-name>  
            <param-value>UTF-8</param-value>  
        </init-param>  
    </filter>  
    <filter-mapping>  
        <filter-name>encodingFilter</filter-name>  
        <url-pattern>/*</url-pattern>  
    </filter-mapping>  
  
    <filter>  
        <filter-name>SSOFilter</filter-name>  
        <filter-class>com.ffcs.sso.filter.SSOFilter</filter-class>  
        <init-param>  
            <!-- 获取票保存的信息 ,并发送单点登陆子站信息  GET方式 -->  
            <param-name>ticketValidateURL</param-name>   
            <param-value>http://127.0.0.1:8080/SSO/webservice/ticket</param-value>  
        </init-param>  
        <init-param>  
            <!-- 定时更新主站用户超时信息，防止子站用户没过期，主站登陆信息已经过期  PUT方式 -->  
            <param-name>updateUserInfoTimeOutURL</param-name>  
            <param-value>http://127.0.0.1:8080/SSO/webservice/userinfo/timeout</param-value>  
        </init-param>  
        <init-param>  
            <!-- 重定向获取 -->  
            <param-name>redirectLoginURL</param-name>  
            <param-value>http://127.0.0.1:8080/SSO/redirect</param-value>  
        </init-param>  
    </filter>  
    <filter-mapping>  
        <filter-name>SSOFilter</filter-name>  
        <url-pattern>/*</url-pattern>  
    </filter-mapping>  
    <servlet>  
        <servlet-name>springMvc</servlet-name>  
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>  
        <init-param>  
            <param-name>contextConfigLocation</param-name>  
            <param-value>classpath:spring-mvc.xml</param-value>  
        </init-param>  
        <!-- <load-on-startup>1</load-on-startup> -->  
    </servlet>  
    <servlet-mapping>  
        <servlet-name>springMvc</servlet-name>  
        <url-pattern>/</url-pattern>  
    </servlet-mapping>  
  
    <!-- 浏览器不支持put,delete等method,由该filter将/blog?_method=delete转换为标准的http delete方法 -->  
    <filter>  
        <filter-name>HiddenHttpMethodFilter</filter-name>  
        <filter-class>org.springframework.web.filter.HiddenHttpMethodFilter</filter-class>  
    </filter>  
  
    <filter-mapping>  
        <filter-name>HiddenHttpMethodFilter</filter-name>  
        <servlet-name>springMvc</servlet-name>  
    </filter-mapping>  
    <error-page>  
        <error-code>404</error-code>  
        <location>/error/404.jsp</location>  
    </error-page>  
    <error-page>  
        <error-code>500</error-code>  
        <location>/error/500.jsp</location>  
    </error-page>  
</web-app>  
```

