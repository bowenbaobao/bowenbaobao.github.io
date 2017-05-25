---
layout: post
title: 支付代码
permalink: /:categories/paycode_url/
date: 2017-5-1 09:30:15 +0800
category: 支付代码
tags: [paycode]
---

### 第三方支付

#### 前端控制

```


package com.sekorm.portal.controller.pay;

import java.io.BufferedReader;
import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.UnsupportedEncodingException;
import java.util.HashMap;
import java.util.Iterator;
import java.util.List;
import java.util.Map;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.dom4j.Document;
import org.dom4j.DocumentException;
import org.dom4j.Element;
import org.dom4j.io.SAXReader;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

import com.sekorm.common.exception.BusinessException;
import com.sekorm.core.common.Constants;
import com.sekorm.core.service.supply.PayDubboService;
import com.sekorm.core.service.trade.OrderService;
import com.sekorm.dubbo.beans.common.DubboReturn;
import com.sekorm.dubbo.beans.isupply.ScanPayResData;
import com.sekorm.dubbo.beans.isupply.WxNotifyDI;
import com.sekorm.portal.common.ShiroUtils;
import com.sekorm.portal.controller.MainController;
import com.sekorm.wxpay.util.ZxingUtils;

/**
 * @describe 跟第三方支付的Controller
 * 
 * @author bowen_bao
 * @date 2017年5月8日
 */

@Controller
@RequestMapping("/pay")
public class PayController extends MainController {

    protected final Logger log = LoggerFactory.getLogger(this.getClass());

	@Autowired
	private PayDubboService payDubboService;
	
	@Autowired
	private OrderService orderService;
	
	@RequestMapping("/aliindex")
	public String aliindex(HttpServletRequest request) {
		return "pay/index";
	}
	
	/**
	 * 支付宝获取支付二维码（下单）
	 * @param request
	 * @param response
	 */
	@RequestMapping(value="/getaliapi", method=RequestMethod.POST)
	public void alipayapi(String orderNo, String dealAmount, 
			HttpServletRequest request, HttpServletResponse response) {
		
		//商户订单号，商户网站订单系统中唯一订单号，必填                                   
//        String out_trade_no = request.getParameter("WIDout_trade_no");
        //订单名称，必填
//        String subject = request.getParameter("WIDsubject");
        //付款金额，必填
//        String total_fee = request.getParameter("WIDtotal_fee");
        //商品描述，可空
//        String body = request.getParameter("WIDbody");
		
		Integer memId = ShiroUtils.getSessionUserId();
		if (memId == null) {
			throw new BusinessException("用户未登录");
		}
		
        String subject = "世强元件电商[" + orderNo + "]";
        String body = "";
		
		try {
			//建立请求
//			log.info("构造跳转到支付宝的html页面开始");
			DubboReturn dubboReturn =  payDubboService.getAliHtml(memId, orderNo, subject, dealAmount, body, getRemortIP(request));
			if (dubboReturn.getCode() == Constants.API_STATUS_OK) {
				String sHtmlText = (String) dubboReturn.getData();
				response.getOutputStream().write(sHtmlText.getBytes());
				log.info("构造跳转到支付宝的html页面成功，sHtmlText："+sHtmlText);
			} else {
				//已经校验了订单金额、是否可以付款等信息
				dubboReturn.getMsg();//失败具体原因
			}
			response.setContentType("text/html charset=utf-8");
		} catch (IOException e) {
			log.error("alipayapi IOException", e);
		}
	}
	
	
	/**
	 * 支付宝同步通知
	 * @param request
	 * @return
	 */
	@RequestMapping("/alireturn")
	public String alireturn(HttpServletRequest request) {
		log.info("portal controller alireture 同步通知*****************************");
		//获取支付宝GET过来反馈信息
		Map<String,String> params = new HashMap<String,String>();
		Map<String, String[]> requestParams = request.getParameterMap();
		for (Iterator<String> iter = requestParams.keySet().iterator(); iter.hasNext();) {
			String name = (String) iter.next();
			String[] values = (String[]) requestParams.get(name);
			String valueStr = "";
			for (int i = 0; i < values.length; i++) {
				valueStr = (i == values.length - 1) ? valueStr + values[i]
						: valueStr + values[i] + ",";
			}
			params.put(name, valueStr);
		}

		//交易状态
		String trade_status = request.getParameter("trade_status");
		
		//支付宝同步验证  TODO 写同步信息到数据库
		boolean verify_result = payDubboService.aliVerify(params); // PORTAL 自己验证
		
		//后台查询订单支付状态
		boolean payStatus= false;
		try {
			payStatus =payDubboService.getpayStatus(params.get("out_trade_no"));
		} catch (Exception e) {
			log.error("支付宝同步查询数据库异常:",e); 
		}
		
		
		if(verify_result){//同步验证成功
			if(trade_status.equals("TRADE_FINISHED") || trade_status.equals("TRADE_SUCCESS")){
				log.info("alireturn TRADE_FINISHED TRADE_SUCCESS:" + params);
				try {
					params.put("memId", "1");//TODO
					params.put("pay_channel", "3");//  3-支付宝扫码
					boolean notify_result = payDubboService.savePayStatus(params);
				} catch (Exception e) {
					log.error("支付宝同步修改订单状态失败:",e); 
				}
				request.setAttribute("result", "alipay return 验证成功");
			}else if(payStatus){
				log.info("alireturn TRADE_FINISHED TRADE_SUCCESS:" + params);
				request.setAttribute("result", "alipay return 验证成功");
			}else{
				log.info("ali alireturn aliVerify fail & payStatus=is no pay " + params);
				request.setAttribute("result", "alipay return 支付正在进行中,请稍后");
			}
			log.info("alireturn verify_result=true:" + params);
//			return "redirect:/trade/order/detail?orderNo=" + params.get("out_trade_no");
		}else{
			log.info("alireturn verify_result=false:" + params);
			request.setAttribute("result", "alipay return 验证失败");
		}
		request.setAttribute("orderNo", params.get("out_trade_no"));
		return "pay/return_url";
	}
	
	
	
	/**
	 * 支付宝异步通知
	 * @param request
	 * @param response
	 */
	@RequestMapping("/alinotify")
	public void alinotify(HttpServletRequest request, HttpServletResponse response) {
		log.info("portal controller alinotify *****************************");
		//获取支付宝POST过来反馈信息
		Map<String,String> params = new HashMap<String,String>();
		Map<String, String[]> requestParams = request.getParameterMap();
		for (Iterator<String> iter = requestParams.keySet().iterator(); iter.hasNext();) {
			String name = (String) iter.next();
			String[] values = (String[]) requestParams.get(name);
			String valueStr = "";
			for (int i = 0; i < values.length; i++) {
				valueStr = (i == values.length - 1) ? valueStr + values[i]
						: valueStr + values[i] + ",";
			}
			//乱码解决，这段代码在出现乱码时使用。如果mysign和sign不相等也可以使用这段代码转化
			//valueStr = new String(valueStr.getBytes("ISO-8859-1"), "gbk");
			params.put(name, valueStr);
		}
		
		//获取支付宝的通知返回参数，可参考技术文档中页面跳转同步通知参数列表(以下仅供参考)//

		try {
			//商户订单号
//			String out_trade_no = new String(request.getParameter("out_trade_no").getBytes("ISO-8859-1"),"UTF-8");
			//支付宝交易号
//			String trade_no = new String(request.getParameter("trade_no").getBytes("ISO-8859-1"),"UTF-8");
			//交易状态
			String trade_status = request.getParameter("trade_status");
			//获取支付宝的通知返回参数，可参考技术文档中页面跳转同步通知参数列表(以上仅供参考)//
			
			if(payDubboService.alipayNotify(params)){//验证成功
				//////////////////////////////////////////////////////////////////////////////////////////
				//请在这里加上商户的业务逻辑程序代码
				
				//——请根据您的业务逻辑来编写程序（以下代码仅作参考）——
				
				if(trade_status.equals("TRADE_FINISHED")){
					//判断该笔订单是否在商户网站中已经做过处理
					//如果没有做过处理，根据订单号（out_trade_no）在商户网站的订单系统中查到该笔订单的详细，并执行商户的业务程序
					//请务必判断请求时的total_fee、seller_id与通知时获取的total_fee、seller_id为一致的
					//如果有做过处理，不执行商户的业务程序
					log.info("alinotify TRADE_FINISHED:" + params);
					//注意：
					//退款日期超过可退款期限后（如三个月可退款），支付宝系统发送该交易状态通知
				} else if (trade_status.equals("TRADE_SUCCESS")){
					//判断该笔订单是否在商户网站中已经做过处理
					//如果没有做过处理，根据订单号（out_trade_no）在商户网站的订单系统中查到该笔订单的详细，并执行商户的业务程序
					//请务必判断请求时的total_fee、seller_id与通知时获取的total_fee、seller_id为一致的
					//如果有做过处理，不执行商户的业务程序
					log.info("alinotify TRADE_SUCCESS:" + params);
					//注意：
					//付款完成后，支付宝系统发送该交易状态通知
				} else {
					log.info("alinotify trade_status:" + params);
				}
				
				//——请根据您的业务逻辑来编写程序（以上代码仅作参考）——
				//支付成功保存数据库
				
				
				//异步通知3次
				
				if(!payDubboService.getpayStatus(params.get("out_trade_no"))){
					params.put("memId", "1");//TODO
					params.put("pay_channel", "3");//  3-支付宝扫码
					boolean notify_result = payDubboService.savePayStatus(params);
				}
				
				response.getOutputStream().write("success".getBytes());
				log.info("AlipayNotify.verify true params=" + params);
			}else{//验证失败
				response.getOutputStream().write("fail".getBytes());
				log.info("AlipayNotify.verify false params=" + params);
			}
		} catch (UnsupportedEncodingException e) {
			log.error("alinotify:" + params, e);
		} catch (IOException e) {
			log.error("alinotify:" + params, e);
		} catch (Exception e) {
			log.error("alinotify:" + params, e);
		}
	}
	
	@RequestMapping("/wxindex")
	public String wxindex(HttpServletRequest request) {
		return "pay/wxindex";
	}
	
	/**
	 * 微信下单，产生二维码的URL
	 * @param request
	 * @return
	 */
	@RequestMapping("/wxpayapi")
	public String wxUnifiedorder(HttpServletRequest request) {
		try {
			//商户订单号，商户网站订单系统中唯一订单号，必填
	        String out_trade_no = payDubboService.getOrderId("0", "1");
	        
	        
	        //订单名称，必填
	        String body = request.getParameter("WIDbody");
	        //付款金额，必填
	        String total_fee = request.getParameter("WIDtotal_fee");
	        log.info("******************************wxpayapi*************** getwxXml");
//	        InetAddress.getLocalHost().getHostAddress() TODO
	        ScanPayResData resData = payDubboService.getObjectFromXML(out_trade_no,body,total_fee,"127.0.0.1");
	        log.info("******************************wxpayapi*************** getwxXml end");
	    	if(resData == null) {
	    		request.setAttribute("error", "API请求逻辑错误，请仔细检测传过去的每一个参数是否合法，或是看API能否被正常访问");
	    		return "pay/wxerror";
	    	}
	    	if(resData.getReturn_code().equals("FALL")) {
	    		request.setAttribute("error", "API系统返回失败，请检测Post给API的数据是否规范合法"); //系统级参数错误
	    		return "pay/wxerror";
	    	}
	    	//先验证一下数据有没有被第三方篡改，确保安全
	    	if(payDubboService.checkIsSignValidFromResponseString(resData.getRs())) {
	    		if(resData.getResult_code().equals("FALL")) {
	    			String err = "出错，错误码：" + resData.getErr_code() + " 错误信息：" + resData.getErr_code_des();
	    			log.error(err);
	    			request.setAttribute("error", err);
	    			return "pay/wxerror";
	    		}
	    		//请求成功，把此记录存到数据库
	    		//现在临时把二维码连接放入session，以后需要存入数据库
	    		//在存入时先把以前的二维码作废，因为同一个订单号可以生成多个支付单号，都可以支付，所以同一个订单号我们要保证只有一个支付链接是可以支付的
//	    		request.getSession().setAttribute("wx_pay_qr_url", resData.getCode_url());
	    		
	    		ShiroUtils.getSession().setAttribute("wx_pay_qr_url", resData.getCode_url()); 
	    		
	    		
	    		request.setAttribute("out_trade_no", out_trade_no);
	    		request.setAttribute("body", body);
	    		request.setAttribute("total_fee", total_fee);
	    		return "pay/wxpay";
	    	} else {
	    		request.setAttribute("error", "API返回的数据签名验证不通过，有可能被第三方篡改!!!");
	    		return "pay/wxerror";
	    	}
		} catch (Exception e) {
			log.error("未知错误", e);
			request.setAttribute("error", e);
		}
		return "pay/wxerror";
	}
	
	
	
	/**
	 * 生产微信二维码图片
	 * @param request
	 * @param response
	 */
	@RequestMapping("/wxpayqr")
	public void wxpayqr(HttpServletRequest request, HttpServletResponse response) {
		try {
			//这里要重数据库里取
//			String wxqrUrl =  (String) request.getSession().getAttribute("wx_pay_qr_url");
			String wxqrUrl =  ShiroUtils.getSession().getAttribute("wx_pay_qr_url").toString();
			if(wxqrUrl != null) {
				response.setContentType("image/png");
				ZxingUtils.setQRCodeImge(wxqrUrl, 430, response.getOutputStream());
			}
			log.info("pay/wxpayqr wxqrUrl=" + wxqrUrl);
		} catch (IOException e) {
			log.error("生成微信支付二维码失败！", e);
		}
	}
	
	
	/**
	 * @discription 微信的异步通知
	 * @param request
	 * @param response
	 */
	@RequestMapping("/wxnotify")
	public void wxnotify(HttpServletRequest request, HttpServletResponse response) {
		log.info("wxnotify ********************** 微信异步通知");
		Map<String, String[]> param = request.getParameterMap();
		StringBuffer sbf = new StringBuffer("wxnotify param:\n");
		
		try {
			sbf = new StringBuffer();
			BufferedReader reader = request.getReader();
			String wxXml;
			while((wxXml = reader.readLine()) != null) {
				sbf.append(wxXml);
			}
			wxXml = sbf.toString();
			log.info("readLine==========="+wxXml);
			
			WxNotifyDI wxNotify=getObjectFromXML(wxXml);
			
			wxNotify.setMemId("1");
			wxNotify.setPay_channel("1");//微信扫码
			
			log.info("wxnotify ********************** 微信异步验证");
			//TODO  校验合法性
			if(payDubboService.checkIsSignValidFromResponseString(wxXml)){
				
				if("SUCCESS".equals(wxNotify.getReturn_code())&& "SUCCESS".equals(wxNotify.getResult_code())) {
					log.info("wxnotify return:SUCCESS OK");
					if(!payDubboService.getpayStatus(wxNotify.getOut_trade_no())){
						boolean notify_result = payDubboService.savePayStatus(wxNotify);
					}				
				}else{
					log.info("wxnotify return:FAIL"+wxNotify.getReturn_code()+"|"+wxNotify.getErr_code_des());
				}
			}else{
				log.info("wxnotify 微信异步校验合法性不通过", "API返回的数据签名验证不通过，有可能被第三方篡改!!!");
			}
		
		} catch (IOException e1) {
			log.error("wxnotify IOException:", e1);
		}
	}
	
	
	/**
	 * @description:web 不断调取订单支付状态	
	 * @param out_trade_no
	 * @return
	 */
	public boolean refreshWxPayStatus(String out_trade_no){
		return payDubboService.getpayStatus(out_trade_no);
	}
	
	
	/**
	 * 微信异步返回xml 转对象
	 * @param xml
	 * @return
	 */
	private WxNotifyDI getObjectFromXML(String xml){
		WxNotifyDI wxNotifyDI=new WxNotifyDI();
		SAXReader reader = new SAXReader();
        try {
        	 
			InputStream is =new ByteArrayInputStream(xml.getBytes("UTF-8"));
        	
			Document document = reader.read(is);
            
            Element root=document.getRootElement(); 

            List list = root.elements(); //得到database元素下的子元素集合

           for(Object obj:list){

           Element element = (Element)obj;
           
           if(element.getName().equals("return_code")){
        	   wxNotifyDI.setReturn_code(element.getText());
           }else if(element.getName().equals("appid")){
        	   wxNotifyDI.setAppid(element.getText());
           }else if(element.getName().equals("mch_id")){
        	   wxNotifyDI.setMch_id(element.getText());
           }else if(element.getName().equals("sign")){
        	   wxNotifyDI.setSign(element.getText());
           }else if(element.getName().equals("result_code")){
        	   wxNotifyDI.setResult_code(element.getText());
           }else if(element.getName().equals("err_code")){
        	   wxNotifyDI.setErr_code(element.getText());
           }else if(element.getName().equals("err_code_des")){
        	   wxNotifyDI.setErr_code_des(element.getText());
           }else if(element.getName().equals("trade_type")){
        	   wxNotifyDI.setTrade_type(element.getText());
           }else if(element.getName().equals("total_fee")){
        	   wxNotifyDI.setTotal_fee(element.getText());
           }else if(element.getName().equals("cash_fee")){
        	   wxNotifyDI.setCash_fee(element.getText());
           }else if(element.getName().equals("transaction_id")){
        	   wxNotifyDI.setTransaction_id(element.getText());
           }else if(element.getName().equals("out_trade_no")){
        	   wxNotifyDI.setOut_trade_no(element.getText());
           }else if(element.getName().equals("time_end")){
        	   wxNotifyDI.setTime_end(element.getText());
           }
           	   wxNotifyDI.setWxXml(xml);
          } 
            
        } catch (DocumentException e) {
            e.printStackTrace();
        } catch (UnsupportedEncodingException e) {
			e.printStackTrace();
		}
		return wxNotifyDI;
	}
	
	
	
	
	
}


```




#### 前端页面

#####  index

```
<%
/* *
 *功能：支付宝即时到账交易接口调试入口页面
 *版本：3.4
 *日期：2016-03-08
 *说明：
 *以下代码只是为了方便商户测试而提供的样例代码，商户可以根据自己网站的需要，按照技术文档编写,并非一定要使用该代码。
 **********************************************
 */
%>
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>支付宝即时到账交易接口</title>
</head>

<style>
    html,body {
        width:100%;
        min-width:1200px;
        height:auto;
        padding:0;
        margin:0;
        font-family:"微软雅黑";
        background-color:#242736
    }
    .header {
        width:100%;
        margin:0 auto;
        height:230px;
        background-color:#fff
    }
    .container {
        width:100%;
        min-width:100px;
        height:auto
    }
    .black {
        background-color:#242736
    }
    .blue {
        background-color:#0ae
    }
    .qrcode {
        width:1200px;
        margin:0 auto;
        height:30px;
        background-color:#242736
    }
    .littlecode {
        width:16px;
        height:16px;
        margin-top:6px;
        cursor:pointer;
        float:right
    }
    .showqrs {
        top:30px;
        position:absolute;
        width:100px;
        margin-left:-65px;
        height:160px;
        display:none
    }
    .shtoparrow {
        width:0;
        height:0;
        margin-left:65px;
        border-left:8px solid transparent;
        border-right:8px solid transparent;
        border-bottom:8px solid #e7e8eb;
        margin-bottom:0;
        font-size:0;
        line-height:0
    }
    .guanzhuqr {
        text-align:center;
        background-color:#e7e8eb;
        border:1px solid #e7e8eb
    }
    .guanzhuqr img {
        margin-top:10px;
        width:80px
    }
    .shmsg {
        margin-left:10px;
        width:80px;
        height:16px;
        line-height:16px;
        font-size:12px;
        color:#242323;
        text-align:center
    }
    .nav {
        width:1200px;
        margin:0 auto;
        height:70px;
    }
    .open,.logo {
        display:block;
        float:left;
        height:40px;
        width:85px;
        margin-top:20px
    }
    .divier {
        display:block;
        float:left;
        margin-left:20px;
        margin-right:20px;
        margin-top:23px;
        width:1px;
        height:24px;
        background-color:#d3d3d3
    }
    .open {
        line-height:30px;
        font-size:20px;
        text-decoration:none;
        color:#1a1a1a
    }
    .navbar {
        float:right;
        width:200px;
        height:40px;
        margin-top:15px;
        list-style:none
    }
    .navbar li {
        float:left;
        width:100px;
        height:40px
    }
    .navbar li a {
        display:inline-block;
        width:100px;
        height:40px;
        line-height:40px;
        font-size:16px;
        color:#1a1a1a;
        text-decoration:none;
        text-align:center
    }
    .navbar li a:hover {
        color:#00AAEE
    }
    .title {
        width:1200px;
        margin:0 auto;
        height:80px;
        line-height:80px;
        font-size:20px;
        color:#FFF
    }
    .content {
        width:100%;
        min-width:1200px;
        height:660px;
        background-color:#fff;      
    }
    .alipayform {
        width:800px;
        margin:0 auto;
        height:600px;
        border:1px solid #0ae
    }
    .element {
        width:600px;
        height:80px;
        margin-left:100px;
        font-size:20px
    }
    .etitle,.einput {
        float:left;
        height:26px
    }
    .etitle {
        width:150px;
        line-height:26px;
        text-align:right
    }
    .einput {
        width:200px;
        margin-left:20px
    }
    .einput input {
        width:398px;
        height:24px;
        border:1px solid #0ae;
        font-size:16px
    }
    .mark {
        margin-top: 10px;
        width:500px;
        height:30px;
        margin-left:80px;
        line-height:30px;
        font-size:12px;
        color:#999
    }
    .legend {
        margin-left:100px;
        font-size:24px
    }
    .alisubmit {
        width:400px;
        height:40px;
        border:0;
        background-color:#0ae;
        font-size:16px;
        color:#FFF;
        cursor:pointer;
        margin-left:170px
    }
    .footer {
        width:100%;
        height:120px;
        background-color:#242735
    }
    .footer-sub a,span {
        color:#808080;
        font-size:12px;
        text-decoration:none
    }
    .footer-sub a:hover {
        color:#00aeee
    }
    .footer-sub span {
        margin:0 3px
    }
    .footer-sub {
        padding-top:40px;
        height:20px;
        width:600px;
        margin:0 auto;
        text-align:center
    }
</style>
<body>
    <div class="header">
        <div class="container black">
            <div class="qrcode">
                <div class="littlecode">
                    <img width="16px" src="img/little_qrcode.jpg" id="licode">
                    <div class="showqrs" id="showqrs">
                        <div class="shtoparrow"></div>
                        <div class="guanzhuqr">
                            <div class="shmsg" style="margin-top:5px;">
                            请扫码关注
                            </div>
                            <div class="shmsg" style="margin-bottom:5px;">
                                接收重要信息
                            </div>
                        </div>
                    </div>
                </div>      
            </div>
        </div>
        <div class="container">
            <div class="nav">
                <a href="https://www.alipay.com/" class="logo"><img src="img/alipay_logo.png" height="30px"></a>
                <span class="divier"></span>
                <a href="http://open.alipay.com/platform/home.htm" class="open" target="_blank">开放平台</a>
                <ul class="navbar">
                    <li><a href="https://doc.open.alipay.com/doc2/detail?treeId=62&articleId=103566&docType=1" target="_blank">在线文档</a></li>
                    <li><a href="https://cschannel.alipay.com/portal.htm?sourceId=213" target="_blank">技术支持</a></li>
                </ul>
            </div>
        </div>
        <div class="container blue">
            <div class="title">支付宝即时到账(create_direct_pay_by_user)</div>
        </div>
    </div>
    <div class="content">
        <form action="/pay/getaliapi" class="alipayform" method="POST" target="_blank">
            <div class="element" style="margin-top:60px;">
                <div class="legend">支付宝即时到账交易接口快速通道 </div>
            </div>
            <div class="element">
                <div class="etitle">商户订单号:</div>
                <div class="einput"><input type="text" name="WIDout_trade_no" id="out_trade_no"></div>
                <br>
                <div class="mark">注意：商户订单号(out_trade_no).必填(建议是英文字母和数字,不能含有特殊字符)</div>
            </div>
            
            <div class="element">
                <div class="etitle">商品名称:</div>
                <div class="einput"><input type="text" name="WIDsubject" value="test商品123"></div>
                <br>
                <div class="mark">注意：产品名称(subject)，必填(建议中文，英文，数字，不能含有特殊字符)</div>
            </div>
            <div class="element">
                <div class="etitle">付款金额:</div>
                <div class="einput"><input type="text" name="WIDtotal_fee" value="0.01"></div>
                <br>
                <div class="mark">注意：付款金额(total_fee)，必填(格式如：1.00,请精确到分)</div>
            </div>
			<div class="element">
                <div class="etitle">商品描述:</div>
                <div class="einput"><input type="text" name="WIDbody" value="即时到账测试"></div>
                <br>
                <div class="mark">注意：商品描述(body)，选填(建议中文，英文，数字，不能含有特殊字符)</div>
            </div>
            <div class="element">
                <input type="submit" class="alisubmit" value ="确认支付">
            </div>
        </form>
    </div>
    <div class="footer">
        <p class="footer-sub">
            <a href="http://ab.alipay.com/i/index.htm" target="_blank">关于支付宝</a><span>|</span>
            <a href="https://e.alipay.com/index.htm" target="_blank">商家中心</a><span>|</span>
            <a href="https://job.alibaba.com/zhaopin/index.htm" target="_blank">诚征英才</a><span>|</span>
            <a href="http://ab.alipay.com/i/lianxi.htm" target="_blank">联系我们</a><span>|</span>
            <a href="#" id="international" target="_blank">International&nbsp;Business</a><span>|</span>
            <a href="http://ab.alipay.com/i/jieshao.htm#en" target="_blank">About Alipay</a>
            <br>
             <span>支付宝版权所有</span>
            <span class="footer-date">2004-2016</span>
            <span><a href="http://fun.alipay.com/certificate/jyxkz.htm" target="_blank">ICP证：沪B2-20150087</a></span>
        </p>

           
    </div>
</body>
<script>

        var even = document.getElementById("licode");   
        var showqrs = document.getElementById("showqrs");
         even.onmouseover = function(){
            showqrs.style.display = "block"; 
         }
         even.onmouseleave = function(){
            showqrs.style.display = "none";
         }
         
         var out_trade_no = document.getElementById("out_trade_no");

         //设定时间格式化函数
         Date.prototype.format = function (format) {
               var args = {
                   "M+": this.getMonth() + 1,
                   "d+": this.getDate(),
                   "h+": this.getHours(),
                   "m+": this.getMinutes(),
                   "s+": this.getSeconds(),
               };
               if (/(y+)/.test(format))
                   format = format.replace(RegExp.$1, (this.getFullYear() + "").substr(4 - RegExp.$1.length));
               for (var i in args) {
                   var n = args[i];
                   if (new RegExp("(" + i + ")").test(format))
                       format = format.replace(RegExp.$1, RegExp.$1.length == 1 ? n : ("00" + n).substr(("" + n).length));
               }
               return format;
           };
           
         out_trade_no.value = 'test'+ new Date().format("yyyyMMddhhmmss");
 
</script>

</html>

```


#####  return_url

```

<!-- 
 功能：支付宝页面跳转同步通知页面
 版本：3.2
 日期：2011-03-17
 说明：
 以下代码只是为了方便商户测试而提供的样例代码，商户可以根据自己网站的需要，按照技术文档编写,并非一定要使用该代码。
 该代码仅供学习和研究支付宝接口使用，只是提供一个参考。

***********页面功能说明***********
 该页面可在本机电脑测试
 可放入HTML等美化页面的代码、商户业务逻辑程序代码
 TRADE_FINISHED(表示交易已经成功结束，并不能再对该交易做后续操作);
 TRADE_SUCCESS(表示交易已经成功结束，可以对该交易做后续操作，如：分润、退款等);
********************************
-->
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>
<html>
  <head>
		<meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
		<title>支付宝页面跳转同步通知页面</title>
  </head>
  <body>
	<h1>${result}</h1>
	<h1>查看订单详情: <a target="_blank" href="${cxtPath }/trade/order/detail?orderNo=${orderNo }">${orderNo }</a></h1>
  </body>
</html>

```



#####  wxerror

```

<!-- 
 功能：支付宝页面跳转同步通知页面
 版本：3.2
 日期：2011-03-17
 说明：
 以下代码只是为了方便商户测试而提供的样例代码，商户可以根据自己网站的需要，按照技术文档编写,并非一定要使用该代码。
 该代码仅供学习和研究支付宝接口使用，只是提供一个参考。

***********页面功能说明***********
 该页面可在本机电脑测试
 可放入HTML等美化页面的代码、商户业务逻辑程序代码
 TRADE_FINISHED(表示交易已经成功结束，并不能再对该交易做后续操作);
 TRADE_SUCCESS(表示交易已经成功结束，可以对该交易做后续操作，如：分润、退款等);
********************************
-->
<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>
<html>
  <head>
		<meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
		<title>微信预支付请求错误页面</title>
  </head>
  <body>
	<h1>${error}</h1>
  </body>
</html>

```

#####  wxindex

```

<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>微信预支付订单请求前页面</title>
</head>

<style>
    html,body {
        width:100%;
        min-width:1200px;
        height:auto;
        padding:0;
        margin:0;
        font-family:"微软雅黑";
        background-color:#242736
    }
    .header {
        width:100%;
        margin:0 auto;
        height:230px;
        background-color:#fff
    }
    .container {
        width:100%;
        min-width:100px;
        height:auto
    }
    .black {
        background-color:#242736
    }
    .blue {
        background-color:#0ae
    }
    .qrcode {
        width:1200px;
        margin:0 auto;
        height:30px;
        background-color:#242736
    }
    .littlecode {
        width:16px;
        height:16px;
        margin-top:6px;
        cursor:pointer;
        float:right
    }
    .showqrs {
        top:30px;
        position:absolute;
        width:100px;
        margin-left:-65px;
        height:160px;
        display:none
    }
    .shtoparrow {
        width:0;
        height:0;
        margin-left:65px;
        border-left:8px solid transparent;
        border-right:8px solid transparent;
        border-bottom:8px solid #e7e8eb;
        margin-bottom:0;
        font-size:0;
        line-height:0
    }
    .guanzhuqr {
        text-align:center;
        background-color:#e7e8eb;
        border:1px solid #e7e8eb
    }
    .guanzhuqr img {
        margin-top:10px;
        width:80px
    }
    .shmsg {
        margin-left:10px;
        width:80px;
        height:16px;
        line-height:16px;
        font-size:12px;
        color:#242323;
        text-align:center
    }
    .nav {
        width:1200px;
        margin:0 auto;
        height:70px;
    }
    .open,.logo {
        display:block;
        float:left;
        height:40px;
        width:85px;
        margin-top:20px
    }
    .divier {
        display:block;
        float:left;
        margin-left:20px;
        margin-right:20px;
        margin-top:23px;
        width:1px;
        height:24px;
        background-color:#d3d3d3
    }
    .open {
        line-height:30px;
        font-size:20px;
        text-decoration:none;
        color:#1a1a1a
    }
    .navbar {
        float:right;
        width:200px;
        height:40px;
        margin-top:15px;
        list-style:none
    }
    .navbar li {
        float:left;
        width:100px;
        height:40px
    }
    .navbar li a {
        display:inline-block;
        width:100px;
        height:40px;
        line-height:40px;
        font-size:16px;
        color:#1a1a1a;
        text-decoration:none;
        text-align:center
    }
    .navbar li a:hover {
        color:#00AAEE
    }
    .title {
        width:1200px;
        margin:0 auto;
        height:80px;
        line-height:80px;
        font-size:20px;
        color:#FFF
    }
    .content {
        width:100%;
        min-width:1200px;
        height:660px;
        background-color:#fff;      
    }
    .alipayform {
        width:800px;
        margin:0 auto;
        height:600px;
        border:1px solid #0ae
    }
    .element {
        width:600px;
        height:80px;
        margin-left:100px;
        font-size:20px
    }
    .etitle,.einput {
        float:left;
        height:26px
    }
    .etitle {
        width:150px;
        line-height:26px;
        text-align:right
    }
    .einput {
        width:200px;
        margin-left:20px
    }
    .einput input {
        width:398px;
        height:24px;
        border:1px solid #0ae;
        font-size:16px
    }
    .mark {
        margin-top: 10px;
        width:500px;
        height:30px;
        margin-left:80px;
        line-height:30px;
        font-size:12px;
        color:#999
    }
    .legend {
        margin-left:100px;
        font-size:24px
    }
    .alisubmit {
        width:400px;
        height:40px;
        border:0;
        background-color:#0ae;
        font-size:16px;
        color:#FFF;
        cursor:pointer;
        margin-left:170px
    }
    .footer {
        width:100%;
        height:120px;
        background-color:#242735
    }
    .footer-sub a,span {
        color:#808080;
        font-size:12px;
        text-decoration:none
    }
    .footer-sub a:hover {
        color:#00aeee
    }
    .footer-sub span {
        margin:0 3px
    }
    .footer-sub {
        padding-top:40px;
        height:20px;
        width:600px;
        margin:0 auto;
        text-align:center
    }
</style>
<body>
    <div class="header">
        <div class="container black">
            <div class="qrcode">
                <div class="littlecode">
                    <img width="16px" src="img/little_qrcode.jpg" id="licode">
                    <div class="showqrs" id="showqrs">
                        <div class="shtoparrow"></div>
                        <div class="guanzhuqr">
                            <div class="shmsg" style="margin-top:5px;">
                            请扫码关注
                            </div>
                            <div class="shmsg" style="margin-bottom:5px;">
                                接收重要信息
                            </div>
                        </div>
                    </div>
                </div>      
            </div>
        </div>
        <div class="container">
            <div class="nav">
                <span class="divier"></span>
            </div>
        </div>
        <div class="container blue">
            <div class="title">微信扫码支付(web)</div>
        </div>
    </div>
    <div class="content">
        <form action="wxpayapi" class="alipayform" method="POST">
            <div class="element" style="margin-top:60px;">
                <div class="legend">填写请求微信预下单api的商品信息</div>
            </div>
            <div class="element">
                <div class="etitle">商户订单号:</div>
                <div class="einput"><input type="text" name="WIDout_trade_no" id="out_trade_no"></div>
                <br>
                <div class="mark">注意：商户订单号(out_trade_no).必填(建议是英文字母和数字,不能含有特殊字符)</div>
            </div>
            
            <div class="element">
                <div class="etitle">商品描述:</div>
                <div class="einput"><input type="text" name="WIDbody" value="test商品123"></div>
                <br>
                <div class="mark">注意：商品描述(body)，必填(建议中文，英文，数字，不能含有特殊字符)</div>
            </div>
            <div class="element">
                <div class="etitle">付款金额:</div>
                <div class="einput"><input type="text" name="WIDtotal_fee" value="1"></div>
                <br>
                <div class="mark">注意：付款金额(total_fee)，必填(格式如：1, 以分为单位)</div>
            </div>
            <div class="element">
                <input type="submit" class="alisubmit" value ="确认支付">
            </div>
        </form>
    </div>
    <div class="footer">
        <p class="footer-sub">
            <a href="http://ab.alipay.com/i/index.htm" target="_blank">关于支付宝</a><span>|</span>
            <a href="https://e.alipay.com/index.htm" target="_blank">商家中心</a><span>|</span>
            <a href="https://job.alibaba.com/zhaopin/index.htm" target="_blank">诚征英才</a><span>|</span>
            <a href="http://ab.alipay.com/i/lianxi.htm" target="_blank">联系我们</a><span>|</span>
            <a href="#" id="international" target="_blank">International&nbsp;Business</a><span>|</span>
            <a href="http://ab.alipay.com/i/jieshao.htm#en" target="_blank">About Alipay</a>
            <br>
             <span>支付宝版权所有</span>
            <span class="footer-date">2004-2016</span>
            <span><a href="http://fun.alipay.com/certificate/jyxkz.htm" target="_blank">ICP证：沪B2-20150087</a></span>
        </p>

           
    </div>
</body>
<script>

        var even = document.getElementById("licode");   
        var showqrs = document.getElementById("showqrs");
         even.onmouseover = function(){
            showqrs.style.display = "block"; 
         }
         even.onmouseleave = function(){
            showqrs.style.display = "none";
         }
         
         var out_trade_no = document.getElementById("out_trade_no");

         //设定时间格式化函数
         Date.prototype.format = function (format) {
               var args = {
                   "M+": this.getMonth() + 1,
                   "d+": this.getDate(),
                   "h+": this.getHours(),
                   "m+": this.getMinutes(),
                   "s+": this.getSeconds(),
               };
               if (/(y+)/.test(format))
                   format = format.replace(RegExp.$1, (this.getFullYear() + "").substr(4 - RegExp.$1.length));
               for (var i in args) {
                   var n = args[i];
                   if (new RegExp("(" + i + ")").test(format))
                       format = format.replace(RegExp.$1, RegExp.$1.length == 1 ? n : ("00" + n).substr(("" + n).length));
               }
               return format;
           };
           
         out_trade_no.value = 'test'+ new Date().format("yyyyMMddhhmmss");
 
</script>

</html>

```





#####  wxpay

```

<%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>微信预支付后的扫码支付页面</title>
</head>

<style>
    html,body {
        width:100%;
        min-width:1200px;
        height:auto;
        padding:0;
        margin:0;
        font-family:"微软雅黑";
        background-color:#242736
    }
    .header {
        width:100%;
        margin:0 auto;
        height:230px;
        background-color:#fff
    }
    .container {
        width:100%;
        min-width:100px;
        height:auto
    }
    .black {
        background-color:#242736
    }
    .blue {
        background-color:#0ae
    }
    .qrcode {
        width:1200px;
        margin:0 auto;
        height:30px;
        background-color:#242736
    }
    .littlecode {
        width:16px;
        height:16px;
        margin-top:6px;
        cursor:pointer;
        float:right
    }
    .showqrs {
        top:30px;
        position:absolute;
        width:100px;
        margin-left:-65px;
        height:160px;
        display:none
    }
    .shtoparrow {
        width:0;
        height:0;
        margin-left:65px;
        border-left:8px solid transparent;
        border-right:8px solid transparent;
        border-bottom:8px solid #e7e8eb;
        margin-bottom:0;
        font-size:0;
        line-height:0
    }
    .guanzhuqr {
        text-align:center;
        background-color:#e7e8eb;
        border:1px solid #e7e8eb
    }
    .guanzhuqr img {
        margin-top:10px;
        width:80px
    }
    .shmsg {
        margin-left:10px;
        width:80px;
        height:16px;
        line-height:16px;
        font-size:12px;
        color:#242323;
        text-align:center
    }
    .nav {
        width:1200px;
        margin:0 auto;
        height:70px;
    }
    .open,.logo {
        display:block;
        float:left;
        height:40px;
        width:85px;
        margin-top:20px
    }
    .divier {
        display:block;
        float:left;
        margin-left:20px;
        margin-right:20px;
        margin-top:23px;
        width:1px;
        height:24px;
        background-color:#d3d3d3
    }
    .open {
        line-height:30px;
        font-size:20px;
        text-decoration:none;
        color:#1a1a1a
    }
    .navbar {
        float:right;
        width:200px;
        height:40px;
        margin-top:15px;
        list-style:none
    }
    .navbar li {
        float:left;
        width:100px;
        height:40px
    }
    .navbar li a {
        display:inline-block;
        width:100px;
        height:40px;
        line-height:40px;
        font-size:16px;
        color:#1a1a1a;
        text-decoration:none;
        text-align:center
    }
    .navbar li a:hover {
        color:#00AAEE
    }
    .title {
        width:1200px;
        margin:0 auto;
        height:80px;
        line-height:80px;
        font-size:20px;
        color:#FFF
    }
    .content {
        width:100%;
        min-width:1200px;
        height:660px;
        background-color:#fff;      
    }
    .alipayform {
        width:800px;
        margin:0 auto;
        height:600px;
        border:1px solid #0ae
    }
    .element {
        width:600px;
        height:80px;
        margin-left:100px;
        font-size:20px
    }
    .etitle,.einput {
        float:left;
        height:26px
    }
    .etitle {
        width:150px;
        line-height:26px;
        text-align:right
    }
    .einput {
        width:200px;
        margin-left:20px
    }
    .einput input {
        width:398px;
        height:24px;
        border:1px solid #0ae;
        font-size:16px
    }
    .mark {
        margin-top: 10px;
        width:500px;
        height:30px;
        margin-left:80px;
        line-height:30px;
        font-size:12px;
        color:#999
    }
    .legend {
        margin-left:100px;
        font-size:24px
    }
    .alisubmit {
        width:400px;
        height:40px;
        border:0;
        background-color:#0ae;
        font-size:16px;
        color:#FFF;
        cursor:pointer;
        margin-left:170px
    }
    .footer {
        width:100%;
        height:120px;
        background-color:#242735
    }
    .footer-sub a,span {
        color:#808080;
        font-size:12px;
        text-decoration:none
    }
    .footer-sub a:hover {
        color:#00aeee
    }
    .footer-sub span {
        margin:0 3px
    }
    .footer-sub {
        padding-top:40px;
        height:20px;
        width:600px;
        margin:0 auto;
        text-align:center
    }
</style>
<body>
    <div class="header">
        <div class="container black">
            <div class="qrcode">
                <div class="littlecode">
                    <img width="16px" src="img/little_qrcode.jpg" id="licode">
                    <div class="showqrs" id="showqrs">
                        <div class="shtoparrow"></div>
                        <div class="guanzhuqr">
                            <div class="shmsg" style="margin-top:5px;">
                            请扫码关注
                            </div>
                            <div class="shmsg" style="margin-bottom:5px;">
                                接收重要信息
                            </div>
                        </div>
                    </div>
                </div>      
            </div>
        </div>
        <div class="container">
            <div class="nav">
                <span class="divier"></span>
            </div>
        </div>
        <div class="container blue">
            <div class="title">微信扫码支付(web)</div>
        </div>
    </div>
    <div class="content">
   	<h2>微信扫码支付</h2>
   	<p>商品描述：${body}</p>
   	<p>订单号：${out_trade_no}</p>
   	<p>总金额：${total_fee}</p>
    <img src="wxpayqr" width="430">
    </div>
    <div class="footer">
        <p class="footer-sub">
            <a href="http://ab.alipay.com/i/index.htm" target="_blank">关于支付宝</a><span>|</span>
            <a href="https://e.alipay.com/index.htm" target="_blank">商家中心</a><span>|</span>
            <a href="https://job.alibaba.com/zhaopin/index.htm" target="_blank">诚征英才</a><span>|</span>
            <a href="http://ab.alipay.com/i/lianxi.htm" target="_blank">联系我们</a><span>|</span>
            <a href="#" id="international" target="_blank">International&nbsp;Business</a><span>|</span>
            <a href="http://ab.alipay.com/i/jieshao.htm#en" target="_blank">About Alipay</a>
            <br>
             <span>支付宝版权所有</span>
            <span class="footer-date">2004-2016</span>
            <span><a href="http://fun.alipay.com/certificate/jyxkz.htm" target="_blank">ICP证：沪B2-20150087</a></span>
        </p>

           
    </div>
</body>
</html>


```

#### 第三方包

```

<!-- pay dependency -->
		<dependency>
			<groupId>com.thoughtworks.xstream</groupId>
			<artifactId>xstream</artifactId>
			<version>1.4.7</version>
		</dependency>
		<dependency>
			<groupId>com.google.zxing</groupId>
			<artifactId>core</artifactId>
			<version>3.3.0</version>
		</dependency>
		<dependency>
			<groupId>commons-httpclient</groupId>
			<artifactId>commons-httpclient</artifactId>
			<version>3.1</version>
		</dependency>
<!-- end -->

```


#### 后台控制

##### PayService

```

package com.sekorm.core.service.supply;

import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.UnsupportedEncodingException;
import java.math.BigDecimal;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import javax.annotation.Resource;
import javax.xml.parsers.ParserConfigurationException;

import org.dom4j.Document;
import org.dom4j.DocumentException;
import org.dom4j.Element;
import org.dom4j.io.SAXReader;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.xml.sax.SAXException;

import com.alipay.api.AlipayApiException;
import com.alipay.api.AlipayClient;
import com.alipay.api.DefaultAlipayClient;
import com.alipay.api.domain.AlipayTradeAppPayModel;
import com.alipay.api.request.AlipayTradeAppPayRequest;
import com.alipay.api.response.AlipayTradeAppPayResponse;
import com.sekorm.common.exception.BusinessException;
import com.sekorm.common.util.JacksonUtils;
import com.sekorm.core.common.PayConstants;
import com.sekorm.core.common.ShopConstants;
import com.sekorm.core.dao.supply.OrderMapper;
import com.sekorm.core.dao.supply.PaymentLogMapper;
import com.sekorm.core.dao.supply.PaymentMapper;
import com.sekorm.core.model.supply.Order;
import com.sekorm.core.model.supply.Payment;
import com.sekorm.core.model.supply.PaymentLog;
import com.sekorm.core.util.ali.config.AlipayConfig;
import com.sekorm.core.util.ali.util.AlipayNotify;
import com.sekorm.core.util.ali.util.AlipaySubmit;
import com.sekorm.core.util.wx.request.ScanPayReqData;
import com.sekorm.core.util.wx.service.ScanPayService;
import com.sekorm.core.util.wx.util.Configure;
import com.sekorm.core.util.wx.util.Signature;
import com.sekorm.core.util.wx.util.Util;
import com.sekorm.dubbo.beans.isupply.PayAliDO;
import com.sekorm.dubbo.beans.isupply.PayWxXmlDO;
import com.sekorm.dubbo.beans.isupply.ScanPayResData;
import com.sekorm.dubbo.beans.isupply.WxNotifyDI;

/**
 * @describe 支付Service
 * @author bowen_bao
 * @date 2017年5月8日
 */
@Service
public class PayService {
	
	private final Logger log = LoggerFactory.getLogger(getClass());
	
	@Resource
	private PaymentMapper paymentMapper;
	@Resource
	private PaymentLogMapper paymentLogMapper;
	@Resource
	private OrderMapper orderMapper;
	
	/**
	 * 检查订单是否可以支付
	 * @author worley_qu
	 * @param dealAmount 客户端传来的订单总金额
	 * @param orderCode  客户端传过来的订单号
	 * @param memId      会员ID
	 */
	private void checkOrderPayment(String dealAmount, String orderCode, Integer memId) {
		Order order = orderMapper.getByOrderCode(orderCode);
		if(order == null || order.getMemId().compareTo(memId) != 0) {
			log.error("用户去支付的订单号不存在, memId=" + memId + ", orderCode=" + orderCode);
			throw new BusinessException("订单号不存在");
		} else if(order.getOrderStatus().intValue() == ShopConstants.ORDER_CANCELED) {
			log.error("该订单已取消，不允许支付， memId=" + memId + ", orderCode=" + orderCode);
			throw new BusinessException("订单已取消");
		} else if(order.getPayStatus().intValue() == ShopConstants.ORDER_PAID) {
			log.error("该订单已完成支付，不能重复支付， memId=" + memId + ", orderCode=" + orderCode);
			throw new BusinessException("订单已支付");
		}
		BigDecimal clientTotalAmount;//客户端传来的订单总金额
		try {
			clientTotalAmount = new BigDecimal(dealAmount);
		} catch (Exception e) {
			log.error("客户端传来的订单总金额错误， memId=" + memId + ", dealAmount=" + dealAmount);
			throw new BusinessException("金额错误");
		}
		if(!order.getDealAmount().equals(clientTotalAmount)) {
			log.error("客户端传来的订单总金额与服务器的不一致， memId=" + memId 
					+ ", 客户端总金额=" + dealAmount
					+ ", 服务端总金额=" + order.getDealAmount());
			throw new BusinessException("金额错误");
		}
	}
	
	/**
	 * 检查订单是否已经成功支付
	 * @param orderCode
	 * @return
	 */
	public boolean checkPayment(String orderCode) {
		Payment payment = paymentMapper.selectByOrderCode(orderCode);
		return payment != null && payment.getStatus() == ShopConstants.ORDER_PAID;
	}
	
	/**
	 * 构造跳转到支付宝的html页面
	 * @param memId
	 * @param orderCode
	 * @param subject
	 * @param dealAmount
	 * @param body
	 * @param ip
	 * @return
	 */
	public String getAliHtml(Integer memId, String orderCode, String subject, 
			String dealAmount, String body, String ip){
		//检查是否可以付款
		checkOrderPayment(dealAmount, orderCode, memId);
		//封装支付参数
		Map<String,String> sPara=new HashMap<String, String>();
		sPara.put("out_trade_no", orderCode);//商户订单号，商户网站订单系统中唯一订单号，必填
		sPara.put("subject", subject);//订单名称，必填
		sPara.put("body", body);//商品描述，可空
		sPara.put("total_fee", dealAmount);//订单付款金额，必填
		sPara.put("_input_charset", AlipayConfig.input_charset);
		sPara.put("exter_invoke_ip", ip);
		sPara.put("partner", AlipayConfig.partner);
		sPara.put("seller_id", AlipayConfig.seller_id);
		sPara.put("notify_url", AlipayConfig.notify_url);
		sPara.put("return_url", AlipayConfig.return_url);
		sPara.put("service", AlipayConfig.service);
		sPara.put("payment_type", AlipayConfig.payment_type);
		
		//构造跳转到支付宝的html页面
		String sHtmlText = AlipaySubmit.buildRequest(sPara,"get","确认");
		log.info("构造跳转到支付宝的html页面完成：memId=" + memId + ", orderCode" + orderCode);
		return sHtmlText;
	}
	
	/**
	 * 获取授权以便能APP能调用ali SDK
	 * @param out_trade_no
	 * @param subject
	 * @param total_fee
	 * @param body
	 * @param ip
	 * @return
	 */
	public PayAliDO getAliPermissionForApp(String out_trade_no,String subject,float total_amount,String ip,String pay_channel){
		
		
		if(checkPayment(out_trade_no)){
			PayAliDO payAliDO=new PayAliDO();
			payAliDO.setPayStatus(1);
			return payAliDO;
		}
		
		AlipayClient alipayClient = new DefaultAlipayClient(AlipayConfig.alipay_gateway_new, 
				AlipayConfig.app_id, AlipayConfig.private_key_APP_ALI, "json", AlipayConfig.input_charset, AlipayConfig.public_key_APP_ALI, AlipayConfig.sign_type_APP_ALI);
		//实例化具体API对应的request类,类名称和接口名称对应,当前调用接口名称：alipay.trade.app.pay
		AlipayTradeAppPayRequest request = new AlipayTradeAppPayRequest();
		//SDK已经封装掉了公共参数，这里只需要传入业务参数。以下方法为sdk的model入参方式(model和biz_content同时存在的情况下取biz_content)。
		AlipayTradeAppPayModel model = new AlipayTradeAppPayModel();
		model.setSubject(subject);
		model.setOutTradeNo(out_trade_no);
		model.setTimeoutExpress(AlipayConfig.timeoutExpress_APP);
		model.setTotalAmount(String.valueOf(total_amount));
		model.setProductCode(AlipayConfig.product_code);
		request.setBizModel(model);
		request.setNotifyUrl(AlipayConfig.notify_url);
		try {
	        //这里和普通的接口调用不同，使用的是sdkExecute
	        AlipayTradeAppPayResponse response = alipayClient.sdkExecute(request);
	        PayAliDO payAliDO=new PayAliDO();
	        payAliDO.setOrderInfo(response.getBody());
	        payAliDO.setPayStatus(0);
	        return payAliDO;
	    } catch (AlipayApiException e) {
	        e.printStackTrace();
		}
		return null;
	}
	
	
	/**
	 * 校验支付宝web支付后返回的url参数，即支付宝同步通知验证参数合法性
	 * @param params
	 * @return
	 */
	public boolean aliVerify(Map<String, String> params){
		log.info("supplyCore payService aliVerify *****************************");
		PaymentLog payLog=new PaymentLog();
		payLog.setOrderCode(params.get("out_trade_no"));
		payLog.setTradeNo(params.get("trade_no"));
		payLog.setType(PayConstants.PAYMENTLOG_TYPE_WEB_ALI_VERIFY);
		payLog.setContent(JacksonUtils.toJson(params));
		paymentLogMapper.insertSelective(payLog);
		return AlipayNotify.verify(params);
	}
	
	
	/**
	 * ali 异步通知验证
	 * @param params
	 * @return
	 */
	public boolean alipayNotify(Map<String, String> params){
		log.info("supplyCore payService AlipayNotify *****************************");
		PaymentLog payLog=new PaymentLog();
		payLog.setOrderCode(params.get("out_trade_no"));
		payLog.setTradeNo(params.get("trade_no"));
		payLog.setType(PayConstants.PAYMENTLOG_TYPE_WEB_ALI_NOTIFY);
		payLog.setContent(JacksonUtils.toJson(params));
		paymentLogMapper.insertSelective(payLog);
		return AlipayNotify.verify(params);
	}
	
	/**
	 * 记录APP支付同步返回日志
	 * @param payChannel 支付渠道
	 * @param result     支付返回结果
	 */
	@SuppressWarnings("rawtypes")
	public void saveAppPayReturnInfo(Integer payChannel, String result) {
		//{code=10000, msg=Success, app_id=2017032006303823, auth_app_id=2017032006303823, charset=utf-8, 
		log.info("APP支付同步返回记录日志:payChannel=" + payChannel + ", result=" + result);
		if(payChannel == PayConstants.PAY_CHANNEL_APP_ALI) {
			Map rs = JacksonUtils.toBean(result, Map.class);
			rs = (Map) rs.get("alipay_trade_app_pay_response");
			insertPaymentLog((String)rs.get("out_trade_no"), PayConstants.PAYMENTLOG_TYPE_ALI_APP_RETURN, 
					(String)rs.get("trade_no"), payChannel, result);
		} else { //微信没有订单号信息
			insertPaymentLog(null, PayConstants.PAYMENTLOG_TYPE_WX_APP_RETURN, 
					null, payChannel, result);
		}
	} 
	
	/**
	 * 记录支付通信日志
	 * @param orderCode
	 * @param type
	 * @param tradeNo
	 * @param payChannel
	 * @param content
	 */
	@Transactional
	private void insertPaymentLog(String orderCode, Integer type, 
			String tradeNo, Integer payChannel, String content) {
		PaymentLog payLog = new PaymentLog();
		payLog.setOrderCode(orderCode);
		payLog.setTradeNo(tradeNo);
		payLog.setPayChannel(payChannel);
		payLog.setType(type); //通信类型（支付接口类型）
		payLog.setContent(content); //具体通信内容
		paymentLogMapper.insertPayLog(payLog);
	}
	/**
	 * 异步通知，保存订单{ALI}
	 * @param params
	 * @return
	 */
	@Transactional
	public boolean savePayStatus(Map<String, String> params){
		log.info("supplyCore payService savePayStatus *****************************");
		String orderCode = params.get("out_trade_no").trim(); //订单号
		String tradeStatus = params.get("trade_status"); //支付宝支付成功失败标识码
		Integer payChannel = Integer.parseInt(params.get("pay_channel"));
		//TODO 校验seller_id app_id
		
		Order order = orderMapper.getByOrderCode(orderCode);
		BigDecimal totalFee = new BigDecimal(params.get("total_fee"));
		if(order == null) { //正常同步通知时不会查不到订单信息
			log.error("支付宝异步通知out_trade_no=" + orderCode + ", 但是系统中不存在该订单");
		} else {
			int status = tradeStatus.equals("TRADE_SUCCESS") ? PayConstants.PAYMENT_STATUS_SUCCESS : PayConstants.PAYMENT_STATUS_ERROR;
			Payment payment = paymentMapper.selectByOrderCode(orderCode);
			if(payment == null) {
				payment = new Payment();
				payment.setMemId(order.getMemId());
				payment.setPayAmount(totalFee); //支付总金额
				payment.setOrderCode(orderCode);
				payment.setPayType(PayConstants.PAYMENT_PAY_TYPE_AMOUNT);
				payment.setTradeNo(params.get("trade_no"));
				payment.setPayChannel(payChannel);
				payment.setStatus(status);      //成功、失败状态值
				payment.setRemark(tradeStatus); //成功、失败字符串
				paymentMapper.insertPay(payment);//新增付款记录
				if(status == ShopConstants.ORDER_PAID) {//成功，修改订单为已付款
					orderMapper.orderPaymentOk(orderCode, payChannel);
				}
			} else if(payment.getStatus().intValue() != ShopConstants.ORDER_PAID
					&& status == PayConstants.PAYMENT_STATUS_SUCCESS) {//以前没有成功，现在支付成功时
				payment.setTradeNo(params.get("trade_no"));
				payment.setPayChannel(payChannel);
				payment.setPayAmount(totalFee);
				payment.setStatus(status);
				payment.setRemark(tradeStatus);
				paymentMapper.updateByOrderCode(payment);//更新付款记录
				//修改订单为已付款
				orderMapper.orderPaymentOk(orderCode, payChannel);
			}
			if(!order.getDealAmount().equals(totalFee)) { //TODO 是否发生异常邮件
				log.error("支付宝异步通知支付金额与系统不一致，out_trade_no=" + orderCode 
						+ ", total_fee=" + params.get("total_fee") 
						+ ", dealAmount=" + order.getDealAmount());
			}
		}
		//记录通信日志
		insertPaymentLog(orderCode, PayConstants.PAYMENTLOG_TYPE_ALI_NOTIFY_SAVE, 
				params.get("trade_no"), payChannel, JacksonUtils.toJson(params));
		return true;
	}
	
	
	/**
	 * 异步通知，保存订单{WX}
	 * @param params
	 * @return
	 */
	@Transactional
	public boolean savePayStatus(WxNotifyDI obj){
		log.info("supplyCore payService savePayStatus *****************************");
		String orderCode = obj.getOut_trade_no(); //订单号
		String resultCode = obj.getResult_code(); //微信支付成功失败标识码
		Integer payChannel = Integer.parseInt(obj.getPay_channel());
		Order order = orderMapper.getByOrderCode(orderCode);
		BigDecimal totalFee = new BigDecimal(obj.getTotal_fee());
		if(order == null) { //正常同步通知时不会查不到订单信息
			log.error("支付宝异步通知out_trade_no=" + orderCode + ", 但是系统中不存在该订单");
		} else {
			int status = resultCode.equals("SUCCESS") ? PayConstants.PAYMENT_STATUS_SUCCESS : PayConstants.PAYMENT_STATUS_ERROR;
			Payment payment = paymentMapper.selectByOrderCode(orderCode);
			if(payment == null) {
				payment = new Payment();
				payment.setMemId(order.getMemId());
				payment.setPayAmount(totalFee); //支付总金额
				payment.setOrderCode(orderCode);
				payment.setPayType(PayConstants.PAYMENT_PAY_TYPE_AMOUNT);
				payment.setTradeNo(obj.getTransaction_id());
				payment.setPayChannel(payChannel);
				payment.setStatus(status);      //成功、失败状态值
				payment.setRemark(resultCode);  //成功、失败字符串
				paymentMapper.insertPay(payment);//新增付款记录
				if(status == ShopConstants.ORDER_PAID) {//成功，修改订单为已付款
					orderMapper.orderPaymentOk(orderCode, payChannel);
				}
			} else if(payment.getStatus().intValue() != ShopConstants.ORDER_PAID
					&& status == PayConstants.PAYMENT_STATUS_SUCCESS) {//以前没有成功，现在支付成功时
				payment.setTradeNo(obj.getTransaction_id());
				payment.setPayAmount(totalFee);
				payment.setPayChannel(payChannel);
				payment.setStatus(status);
				payment.setRemark(resultCode);
				paymentMapper.updateByOrderCode(payment);//更新付款记录
				//修改订单为已付款
				orderMapper.orderPaymentOk(orderCode, payChannel);
			}
			if(!order.getDealAmount().equals(totalFee)) { //TODO 是否发生异常邮件
				log.error("微信异步通知支付金额与系统不一致，out_trade_no=" + orderCode 
						+ ", total_fee=" + obj.getTotal_fee() 
						+ ", dealAmount=" + order.getDealAmount());
			}
		}
		//记录通信日志
		insertPaymentLog(orderCode, PayConstants.PAYMENTLOG_TYPE_WEB_WX_NOTIFY_SAVE, 
				obj.getTransaction_id(), payChannel, obj.getWxXml());
		return true;
	}
	
	
	
	/**
	 * @descripition 查询订单的状态是否为已支付
	 * @param out_trade_no 订单号
	 * @return
	 */
//	public boolean getpayStatus(String out_trade_no){
//		return paymentService.checkPayment(out_trade_no);
//	}
	
	/**
	 * 微信WEB统一下单获取二维码URL
	 * @param out_trade_no
	 * @param body
	 * @param total_fee
	 * @param ip
	 * @return
	 */
	public ScanPayResData getObjectFromXML(String out_trade_no, String body,
			String total_fee,String ip){
		log.info("get WX xml  **********************************************");
		ScanPayReqData data = new ScanPayReqData();
        data.setBody(body);
    	data.setOut_trade_no(out_trade_no);
    	data.setTotal_fee(Integer.parseInt(total_fee));
    	data.setSpbill_create_ip(ip);
    	data.setNotify_url(Configure.notify_url);
    	data.setTrade_type(Configure.trade_type);
    	data.setSign(Signature.getSign(data.toMap()));
    	
    	ScanPayService service;
		try {
			service = new ScanPayService();
			String rs = service.request(data); //微信返回的xml数据
			ScanPayResData resData = (ScanPayResData) Util.getObjectFromXML(rs, ScanPayResData.class);
			resData.setRs(rs);
			return resData;
		} catch (Exception e) {
			e.printStackTrace();
		}
    	
		return null;
	}
	
	/**
	 * 获取授权以便能APP能调用wx SDK
	 * @param out_trade_no
	 * @param subject
	 * @param total_fee
	 * @param body
	 * @param ip
	 * @return
	 */
	public PayWxXmlDO getWxPermissionForApp(String out_trade_no,String total_fee, String body,
			String ip){
		
		if(checkPayment(out_trade_no)){
			PayWxXmlDO xmlDO=new PayWxXmlDO();
			xmlDO.setPayStatus(1);
			return xmlDO;
		}
		
		ScanPayReqData data = new ScanPayReqData();
		data.setAppid(Configure.appID_app);
		data.setMch_id(Configure.mchID_app);
		data.setDevice_info("APP");
        data.setBody(body);
    	data.setOut_trade_no(out_trade_no);
    	data.setTotal_fee(Integer.parseInt(total_fee));
//    	data.setSpbill_create_ip(ip);//TODO
    	
    	data.setNotify_url(Configure.notify_url);
    	data.setTrade_type(Configure.trade_type_app);
    	data.setSign(Signature.getSignApp(data.toMap()));
    	ScanPayService service;
		try {
			service = new ScanPayService("app");
			String xmlStr=service.request(data,"app");
			System.out.println("xml:"+xmlStr);
			return convertPayWxXmlDO(xmlStr); //微信返回的xml数据;
		} catch (Exception e) {
			e.printStackTrace();
		}
    	
		return null;
	}
	
	/**
	 * 微信APP 统一下单返回的xml 转对象
	 * @param xml
	 * @return
	 */
	public PayWxXmlDO convertPayWxXmlDO(String xml){
		PayWxXmlDO xmlDO=new PayWxXmlDO();
		xmlDO.setKey(Configure.key_app);
		xmlDO.setPayStatus(0);
		SAXReader reader = new SAXReader();
        try {
        	 
			InputStream is =new ByteArrayInputStream(xml.getBytes("UTF-8"));
        	
			Document document = reader.read(is);
            
            Element root=document.getRootElement(); 

            List list = root.elements(); //得到database元素下的子元素集合

           for(Object obj:list){

           Element element = (Element)obj;

           if(element.getName().equals("prepay_id")){
        	   xmlDO.setPrepay_id(element.getText());
           }else if(element.getName().equals("mch_id")){
        	   xmlDO.setMch_id(element.getText());
           }else if(element.getName().equals("result_code")){
        	   xmlDO.setResult_code(element.getText());
           }else if(element.getName().equals("return_msg")){
        	   xmlDO.setReturn_msg(element.getText());
           }
           
          } 
            
        } catch (DocumentException e) {
            e.printStackTrace();
        } catch (UnsupportedEncodingException e) {
			e.printStackTrace();
		}
        
        
		return xmlDO;
	}
	
	  /**
	   * web wx 下单获取二维码URL，校验参数
	   * @param responseString
	   * @return
	   */
	  public boolean checkIsSignValidFromResponseString(String responseString){
		  try {
			return Signature.checkIsSignValidFromResponseString(responseString);
		} catch (ParserConfigurationException e) {
			e.printStackTrace();
		} catch (IOException e) {
			e.printStackTrace();
		} catch (SAXException e) {
			e.printStackTrace();
		}
		  return false;
	  }  
	
}


```


### util

#### ali

##### config

```
package com.sekorm.core.util.ali.config;


/**
 *类名：AlipayConfig
 *功能：基础配置类
 *详细：设置帐户有关信息及返回路径
 *版本：3.4
 *修改日期：2016-03-08
 */
public class AlipayConfig {
	
	public static  String partner;
	
	public static String seller_id;
	
	public static String private_key;
	
	public static   String alipay_public_key;
	
	public static   String notify_url;
	
	public static   String return_url;
	
	public static   String sign_type;
	
	public static   String input_charset;
	
	public static   String service;
	
	public static   String payment_type;
	
	public static   String anti_phishing_key;
	
	public static   String exter_invoke_ip;
	
	public static   String log_path;
	
	public static  String alipay_gateway_new;
	
	//app
	public static String app_id;
	public static String method;
	public static String charset; 
	public static String version; 
	public static String product_code; 
	public static String sign_type_APP_ALI;
	public static String private_key_APP_ALI;
	public static String public_key_APP_ALI;
	public static String timeoutExpress_APP;
	
	public static String getTimeoutExpress_APP() {
		return timeoutExpress_APP;
	}

	public static void setTimeoutExpress_APP(String timeoutExpress_APP) {
		AlipayConfig.timeoutExpress_APP = timeoutExpress_APP;
	}

	public static String getPartner() {
		return partner;
	}

	public static void setPartner(String partner) {
		AlipayConfig.partner = partner;
	}

	public static String getSeller_id() {
		return seller_id;
	}

	public static void setSeller_id(String seller_id) {
		AlipayConfig.seller_id = seller_id;
	}

	public static String getPrivate_key() {
		return private_key;
	}

	public static void setPrivate_key(String private_key) {
		AlipayConfig.private_key = private_key;
	}

	public static String getAlipay_public_key() {
		return alipay_public_key;
	}

	public static void setAlipay_public_key(String alipay_public_key) {
		AlipayConfig.alipay_public_key = alipay_public_key;
	}

	public static String getNotify_url() {
		return notify_url;
	}

	public static void setNotify_url(String notify_url) {
		AlipayConfig.notify_url = notify_url;
	}

	public static String getReturn_url() {
		return return_url;
	}

	public static void setReturn_url(String return_url) {
		AlipayConfig.return_url = return_url;
	}

	public static String getSign_type() {
		return sign_type;
	}

	public static void setSign_type(String sign_type) {
		AlipayConfig.sign_type = sign_type;
	}

	public static String getInput_charset() {
		return input_charset;
	}

	public static void setInput_charset(String input_charset) {
		AlipayConfig.input_charset = input_charset;
	}

	public static String getService() {
		return service;
	}

	public static void setService(String service) {
		AlipayConfig.service = service;
	}

	public static String getPayment_type() {
		return payment_type;
	}

	public static void setPayment_type(String payment_type) {
		AlipayConfig.payment_type = payment_type;
	}

	public static String getAnti_phishing_key() {
		return anti_phishing_key;
	}

	public static void setAnti_phishing_key(String anti_phishing_key) {
		AlipayConfig.anti_phishing_key = anti_phishing_key;
	}

	public static String getExter_invoke_ip() {
		return exter_invoke_ip;
	}

	public static void setExter_invoke_ip(String exter_invoke_ip) {
		AlipayConfig.exter_invoke_ip = exter_invoke_ip;
	}

	public static String getLog_path() {
		return log_path;
	}

	public static void setLog_path(String log_path) {
		AlipayConfig.log_path = log_path;
	}

	public static String getApp_id() {
		return app_id;
	}

	public static void setApp_id(String app_id) {
		AlipayConfig.app_id = app_id;
	}

	public static String getMethod() {
		return method;
	}

	public static void setMethod(String method) {
		AlipayConfig.method = method;
	}

	public static String getCharset() {
		return charset;
	}

	public static void setCharset(String charset) {
		AlipayConfig.charset = charset;
	}

	public static String getVersion() {
		return version;
	}

	public static void setVersion(String version) {
		AlipayConfig.version = version;
	}

	public static String getProduct_code() {
		return product_code;
	}

	public static void setProduct_code(String product_code) {
		AlipayConfig.product_code = product_code;
	}

	public static String getSign_type_APP_ALI() {
		return sign_type_APP_ALI;
	}

	public static void setSign_type_APP_ALI(String sign_type_APP_ALI) {
		AlipayConfig.sign_type_APP_ALI = sign_type_APP_ALI;
	}

	public static String getPrivate_key_APP_ALI() {
		return private_key_APP_ALI;
	}

	public static void setPrivate_key_APP_ALI(String private_key_APP_ALI) {
		AlipayConfig.private_key_APP_ALI = private_key_APP_ALI;
	}

	public static String getPublic_key_APP_ALI() {
		return public_key_APP_ALI;
	}

	public static void setPublic_key_APP_ALI(String public_key_APP_ALI) {
		AlipayConfig.public_key_APP_ALI = public_key_APP_ALI;
	}

	public static String getAlipay_gateway_new() {
		return alipay_gateway_new;
	}

	public static void setAlipay_gateway_new(String alipay_gateway_new) {
		AlipayConfig.alipay_gateway_new = alipay_gateway_new;
	}

	
	
	
	
	
	
}



```


##### sign

```



/*
 * Copyright (C) 2010 The MobileSecurePay Project
 * All right reserved.
 * author: shiqun.shi@alipay.com
 */

package com.sekorm.core.util.ali.sign;

public final class Base64 {

    static private final int     BASELENGTH           = 128;
    static private final int     LOOKUPLENGTH         = 64;
    static private final int     TWENTYFOURBITGROUP   = 24;
    static private final int     EIGHTBIT             = 8;
    static private final int     SIXTEENBIT           = 16;
    static private final int     FOURBYTE             = 4;
    static private final int     SIGN                 = -128;
    static private final char    PAD                  = '=';
    static private final boolean fDebug               = false;
    static final private byte[]  base64Alphabet       = new byte[BASELENGTH];
    static final private char[]  lookUpBase64Alphabet = new char[LOOKUPLENGTH];

    static {
        for (int i = 0; i < BASELENGTH; ++i) {
            base64Alphabet[i] = -1;
        }
        for (int i = 'Z'; i >= 'A'; i--) {
            base64Alphabet[i] = (byte) (i - 'A');
        }
        for (int i = 'z'; i >= 'a'; i--) {
            base64Alphabet[i] = (byte) (i - 'a' + 26);
        }

        for (int i = '9'; i >= '0'; i--) {
            base64Alphabet[i] = (byte) (i - '0' + 52);
        }

        base64Alphabet['+'] = 62;
        base64Alphabet['/'] = 63;

        for (int i = 0; i <= 25; i++) {
            lookUpBase64Alphabet[i] = (char) ('A' + i);
        }

        for (int i = 26, j = 0; i <= 51; i++, j++) {
            lookUpBase64Alphabet[i] = (char) ('a' + j);
        }

        for (int i = 52, j = 0; i <= 61; i++, j++) {
            lookUpBase64Alphabet[i] = (char) ('0' + j);
        }
        lookUpBase64Alphabet[62] = (char) '+';
        lookUpBase64Alphabet[63] = (char) '/';

    }

    private static boolean isWhiteSpace(char octect) {
        return (octect == 0x20 || octect == 0xd || octect == 0xa || octect == 0x9);
    }

    private static boolean isPad(char octect) {
        return (octect == PAD);
    }

    private static boolean isData(char octect) {
        return (octect < BASELENGTH && base64Alphabet[octect] != -1);
    }

    /**
     * Encodes hex octects into Base64
     *
     * @param binaryData Array containing binaryData
     * @return Encoded Base64 array
     */
    public static String encode(byte[] binaryData) {

        if (binaryData == null) {
            return null;
        }

        int lengthDataBits = binaryData.length * EIGHTBIT;
        if (lengthDataBits == 0) {
            return "";
        }

        int fewerThan24bits = lengthDataBits % TWENTYFOURBITGROUP;
        int numberTriplets = lengthDataBits / TWENTYFOURBITGROUP;
        int numberQuartet = fewerThan24bits != 0 ? numberTriplets + 1 : numberTriplets;
        char encodedData[] = null;

        encodedData = new char[numberQuartet * 4];

        byte k = 0, l = 0, b1 = 0, b2 = 0, b3 = 0;

        int encodedIndex = 0;
        int dataIndex = 0;
        if (fDebug) {
            System.out.println("number of triplets = " + numberTriplets);
        }

        for (int i = 0; i < numberTriplets; i++) {
            b1 = binaryData[dataIndex++];
            b2 = binaryData[dataIndex++];
            b3 = binaryData[dataIndex++];

            if (fDebug) {
                System.out.println("b1= " + b1 + ", b2= " + b2 + ", b3= " + b3);
            }

            l = (byte) (b2 & 0x0f);
            k = (byte) (b1 & 0x03);

            byte val1 = ((b1 & SIGN) == 0) ? (byte) (b1 >> 2) : (byte) ((b1) >> 2 ^ 0xc0);
            byte val2 = ((b2 & SIGN) == 0) ? (byte) (b2 >> 4) : (byte) ((b2) >> 4 ^ 0xf0);
            byte val3 = ((b3 & SIGN) == 0) ? (byte) (b3 >> 6) : (byte) ((b3) >> 6 ^ 0xfc);

            if (fDebug) {
                System.out.println("val2 = " + val2);
                System.out.println("k4   = " + (k << 4));
                System.out.println("vak  = " + (val2 | (k << 4)));
            }

            encodedData[encodedIndex++] = lookUpBase64Alphabet[val1];
            encodedData[encodedIndex++] = lookUpBase64Alphabet[val2 | (k << 4)];
            encodedData[encodedIndex++] = lookUpBase64Alphabet[(l << 2) | val3];
            encodedData[encodedIndex++] = lookUpBase64Alphabet[b3 & 0x3f];
        }

        // form integral number of 6-bit groups
        if (fewerThan24bits == EIGHTBIT) {
            b1 = binaryData[dataIndex];
            k = (byte) (b1 & 0x03);
            if (fDebug) {
                System.out.println("b1=" + b1);
                System.out.println("b1<<2 = " + (b1 >> 2));
            }
            byte val1 = ((b1 & SIGN) == 0) ? (byte) (b1 >> 2) : (byte) ((b1) >> 2 ^ 0xc0);
            encodedData[encodedIndex++] = lookUpBase64Alphabet[val1];
            encodedData[encodedIndex++] = lookUpBase64Alphabet[k << 4];
            encodedData[encodedIndex++] = PAD;
            encodedData[encodedIndex++] = PAD;
        } else if (fewerThan24bits == SIXTEENBIT) {
            b1 = binaryData[dataIndex];
            b2 = binaryData[dataIndex + 1];
            l = (byte) (b2 & 0x0f);
            k = (byte) (b1 & 0x03);

            byte val1 = ((b1 & SIGN) == 0) ? (byte) (b1 >> 2) : (byte) ((b1) >> 2 ^ 0xc0);
            byte val2 = ((b2 & SIGN) == 0) ? (byte) (b2 >> 4) : (byte) ((b2) >> 4 ^ 0xf0);

            encodedData[encodedIndex++] = lookUpBase64Alphabet[val1];
            encodedData[encodedIndex++] = lookUpBase64Alphabet[val2 | (k << 4)];
            encodedData[encodedIndex++] = lookUpBase64Alphabet[l << 2];
            encodedData[encodedIndex++] = PAD;
        }

        return new String(encodedData);
    }

    /**
     * Decodes Base64 data into octects
     *
     * @param encoded string containing Base64 data
     * @return Array containind decoded data.
     */
    public static byte[] decode(String encoded) {

        if (encoded == null) {
            return null;
        }

        char[] base64Data = encoded.toCharArray();
        // remove white spaces
        int len = removeWhiteSpace(base64Data);

        if (len % FOURBYTE != 0) {
            return null;//should be divisible by four
        }

        int numberQuadruple = (len / FOURBYTE);

        if (numberQuadruple == 0) {
            return new byte[0];
        }

        byte decodedData[] = null;
        byte b1 = 0, b2 = 0, b3 = 0, b4 = 0;
        char d1 = 0, d2 = 0, d3 = 0, d4 = 0;

        int i = 0;
        int encodedIndex = 0;
        int dataIndex = 0;
        decodedData = new byte[(numberQuadruple) * 3];

        for (; i < numberQuadruple - 1; i++) {

            if (!isData((d1 = base64Data[dataIndex++])) || !isData((d2 = base64Data[dataIndex++]))
                || !isData((d3 = base64Data[dataIndex++]))
                || !isData((d4 = base64Data[dataIndex++]))) {
                return null;
            }//if found "no data" just return null

            b1 = base64Alphabet[d1];
            b2 = base64Alphabet[d2];
            b3 = base64Alphabet[d3];
            b4 = base64Alphabet[d4];

            decodedData[encodedIndex++] = (byte) (b1 << 2 | b2 >> 4);
            decodedData[encodedIndex++] = (byte) (((b2 & 0xf) << 4) | ((b3 >> 2) & 0xf));
            decodedData[encodedIndex++] = (byte) (b3 << 6 | b4);
        }

        if (!isData((d1 = base64Data[dataIndex++])) || !isData((d2 = base64Data[dataIndex++]))) {
            return null;//if found "no data" just return null
        }

        b1 = base64Alphabet[d1];
        b2 = base64Alphabet[d2];

        d3 = base64Data[dataIndex++];
        d4 = base64Data[dataIndex++];
        if (!isData((d3)) || !isData((d4))) {//Check if they are PAD characters
            if (isPad(d3) && isPad(d4)) {
                if ((b2 & 0xf) != 0)//last 4 bits should be zero
                {
                    return null;
                }
                byte[] tmp = new byte[i * 3 + 1];
                System.arraycopy(decodedData, 0, tmp, 0, i * 3);
                tmp[encodedIndex] = (byte) (b1 << 2 | b2 >> 4);
                return tmp;
            } else if (!isPad(d3) && isPad(d4)) {
                b3 = base64Alphabet[d3];
                if ((b3 & 0x3) != 0)//last 2 bits should be zero
                {
                    return null;
                }
                byte[] tmp = new byte[i * 3 + 2];
                System.arraycopy(decodedData, 0, tmp, 0, i * 3);
                tmp[encodedIndex++] = (byte) (b1 << 2 | b2 >> 4);
                tmp[encodedIndex] = (byte) (((b2 & 0xf) << 4) | ((b3 >> 2) & 0xf));
                return tmp;
            } else {
                return null;
            }
        } else { //No PAD e.g 3cQl
            b3 = base64Alphabet[d3];
            b4 = base64Alphabet[d4];
            decodedData[encodedIndex++] = (byte) (b1 << 2 | b2 >> 4);
            decodedData[encodedIndex++] = (byte) (((b2 & 0xf) << 4) | ((b3 >> 2) & 0xf));
            decodedData[encodedIndex++] = (byte) (b3 << 6 | b4);

        }

        return decodedData;
    }

    /**
     * remove WhiteSpace from MIME containing encoded Base64 data.
     *
     * @param data  the byte array of base64 data (with WS)
     * @return      the new length
     */
    private static int removeWhiteSpace(char[] data) {
        if (data == null) {
            return 0;
        }

        // count characters that's not whitespace
        int newSize = 0;
        int len = data.length;
        for (int i = 0; i < len; i++) {
            if (!isWhiteSpace(data[i])) {
                data[newSize++] = data[i];
            }
        }
        return newSize;
    }
}





*******************************************************************************



package com.sekorm.core.util.ali.sign;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.InputStream;
import java.security.KeyFactory;
import java.security.PrivateKey;
import java.security.PublicKey;
import java.security.spec.PKCS8EncodedKeySpec;
import java.security.spec.X509EncodedKeySpec;

import com.sekorm.core.util.ali.config.AlipayConfig;

//import javax.crypto.Cipher;

public class RSA {

	private static final String SIGN_TYPE_RSA = "RSA";
	public static final String SIGN_ALGORITHMS = "SHA1WithRSA";

	private static final String SIGN_TYPE_RSA2 = "RSA2";
	private static final String SIGN_SHA256RSA_ALGORITHMS = "SHA256WithRSA";

	/**
	 * RSA签名
	 * 
	 * @param content
	 *            待签名数据
	 * @param privateKey
	 *            商户私钥
	 * @param input_charset
	 *            编码格式
	 * @param signType
	 *            签名类型      
	 * @return 签名值
	 */
	public static String sign(String content, String privateKey, 
			String input_charset, String signType) {
		try {
			PKCS8EncodedKeySpec priPKCS8 = new PKCS8EncodedKeySpec(
					Base64.decode(privateKey));
			KeyFactory keyf = KeyFactory.getInstance("RSA");
			PrivateKey priKey = keyf.generatePrivate(priPKCS8);

			java.security.Signature signature = null;
			if (SIGN_TYPE_RSA.equals(signType)) {
	            signature = java.security.Signature.getInstance(SIGN_ALGORITHMS);
	        } else if (SIGN_TYPE_RSA2.equals(signType)) {
	            signature = java.security.Signature.getInstance(SIGN_SHA256RSA_ALGORITHMS);
	        } else {
	            throw new Exception("不是支持的签名类型 : signType=" + signType);
	        }

			signature.initSign(priKey);
			signature.update(content.getBytes(input_charset));

			byte[] signed = signature.sign();

			return Base64.encode(signed);
		} catch (Exception e) {
			e.printStackTrace();
		}

		return null;
	}

	/**
	 * RSA验签名检查
	 * 
	 * @param content
	 *            待签名数据
	 * @param sign
	 *            签名值
	 * @param ali_public_key
	 *            支付宝公钥
	 * @param input_charset
	 *            编码格式
	 * @param signType
	 *            签名类型
	 * @return 布尔值
	 */
	public static boolean verify(String content, String sign, 
			String ali_public_key, String input_charset, String signType) {
		try {
			KeyFactory keyFactory = KeyFactory.getInstance("RSA");
			byte[] encodedKey = Base64.decode(ali_public_key);
			PublicKey pubKey = keyFactory
					.generatePublic(new X509EncodedKeySpec(encodedKey));

			java.security.Signature signature = null;
			if (SIGN_TYPE_RSA.equals(signType)) {
	            signature = java.security.Signature.getInstance(SIGN_ALGORITHMS);
	        } else if (SIGN_TYPE_RSA2.equals(signType)) {
	            signature = java.security.Signature.getInstance(SIGN_SHA256RSA_ALGORITHMS);
	        } else {
	            throw new Exception("不是支持的签名类型 : signType=" + signType);
	        }

			signature.initVerify(pubKey);
			signature.update(content.getBytes(input_charset));

			boolean bverify = signature.verify(Base64.decode(sign));
			return bverify;
		} catch (Exception e) {
			e.printStackTrace();
		}
		return false;
	}

	/**
	 * 解密
	 * 
	 * @param content
	 *            密文
	 * @param private_key
	 *            商户私钥
	 * @param input_charset
	 *            编码格式
	 * @return 解密后的字符串
	 */
	public static String decrypt(String content, String private_key, String input_charset) throws Exception {
		PrivateKey prikey = getPrivateKey(private_key);

//		Cipher cipher = Cipher.getInstance("RSA");
//		cipher.init(Cipher.DECRYPT_MODE, prikey);

		InputStream ins = new ByteArrayInputStream(Base64.decode(content));
		ByteArrayOutputStream writer = new ByteArrayOutputStream();
		// rsa解密的字节大小最多是128，将需要解密的内容，按128位拆开解密
		byte[] buf = new byte[128];
		int bufl;

		while ((bufl = ins.read(buf)) != -1) {
			byte[] block = null;

			if (buf.length == bufl) {
				block = buf;
			} else {
				block = new byte[bufl];
				for (int i = 0; i < bufl; i++) {
					block[i] = buf[i];
				}
			}

//			writer.write(cipher.doFinal(block));
		}

		return new String(writer.toByteArray(), input_charset);
	}

	/**
	 * 得到私钥
	 * 
	 * @param key
	 *            密钥字符串（经过base64编码）
	 * @throws Exception
	 */
	public static PrivateKey getPrivateKey(String key) throws Exception {

		byte[] keyBytes;

		keyBytes = Base64.decode(key);

		PKCS8EncodedKeySpec keySpec = new PKCS8EncodedKeySpec(keyBytes);

		KeyFactory keyFactory = KeyFactory.getInstance("RSA");

		PrivateKey privateKey = keyFactory.generatePrivate(keySpec);

		return privateKey;
	}
	
	public static void main(String[] args) {
		String sign = sign("a=1", AlipayConfig.private_key_APP_ALI, "utf-8", "RSA2");
		System.out.println(sign);
		boolean rs = verify("a=1", sign, AlipayConfig.public_key_APP_ALI, "utf-8", "RSA2");
		System.out.println(rs);
	}
	
	
	
}



```


##### util

```

=====================================================================================

###### httpClient

************************************************************************************

package com.sekorm.core.util.ali.util.httpClient;

import org.apache.commons.httpclient.HttpException;
import java.io.IOException;
import java.net.UnknownHostException;

import org.apache.commons.httpclient.HttpClient;
import org.apache.commons.httpclient.HttpConnectionManager;
import org.apache.commons.httpclient.HttpMethod;
import org.apache.commons.httpclient.MultiThreadedHttpConnectionManager;
import org.apache.commons.httpclient.NameValuePair;
import org.apache.commons.httpclient.methods.GetMethod;
import org.apache.commons.httpclient.methods.PostMethod;
import org.apache.commons.httpclient.util.IdleConnectionTimeoutThread;
import org.apache.commons.httpclient.methods.multipart.FilePart;
import org.apache.commons.httpclient.methods.multipart.FilePartSource;
import org.apache.commons.httpclient.methods.multipart.MultipartRequestEntity;
import org.apache.commons.httpclient.methods.multipart.Part;
import org.apache.commons.httpclient.methods.multipart.StringPart;
import org.apache.commons.httpclient.params.HttpMethodParams;
import java.io.File;
import java.util.ArrayList;
import java.util.List;

/* *
 *类名：HttpProtocolHandler
 *功能：HttpClient方式访问
 *详细：获取远程HTTP数据
 *版本：3.3
 *日期：2012-08-17
 *说明：
 *以下代码只是为了方便商户测试而提供的样例代码，商户可以根据自己网站的需要，按照技术文档编写,并非一定要使用该代码。
 *该代码仅供学习和研究支付宝接口使用，只是提供一个参考。
 */

public class HttpProtocolHandler {

    private static String              DEFAULT_CHARSET                     = "UTF-8";

    /** 连接超时时间，由bean factory设置，缺省为8秒钟 */
    private int                        defaultConnectionTimeout            = 8000;

    /** 回应超时时间, 由bean factory设置，缺省为30秒钟 */
    private int                        defaultSoTimeout                    = 30000;

    /** 闲置连接超时时间, 由bean factory设置，缺省为60秒钟 */
    private int                        defaultIdleConnTimeout              = 60000;

    private int                        defaultMaxConnPerHost               = 30;

    private int                        defaultMaxTotalConn                 = 80;

    /** 默认等待HttpConnectionManager返回连接超时（只有在达到最大连接数时起作用）：1秒*/
    private static final long          defaultHttpConnectionManagerTimeout = 3 * 1000;

    /**
     * HTTP连接管理器，该连接管理器必须是线程安全的.
     */
    private HttpConnectionManager      connectionManager;

    private static HttpProtocolHandler httpProtocolHandler                 = new HttpProtocolHandler();

    /**
     * 工厂方法
     * 
     * @return
     */
    public static HttpProtocolHandler getInstance() {
        return httpProtocolHandler;
    }

    /**
     * 私有的构造方法
     */
    private HttpProtocolHandler() {
        // 创建一个线程安全的HTTP连接池
        connectionManager = new MultiThreadedHttpConnectionManager();
        connectionManager.getParams().setDefaultMaxConnectionsPerHost(defaultMaxConnPerHost);
        connectionManager.getParams().setMaxTotalConnections(defaultMaxTotalConn);

        IdleConnectionTimeoutThread ict = new IdleConnectionTimeoutThread();
        ict.addConnectionManager(connectionManager);
        ict.setConnectionTimeout(defaultIdleConnTimeout);

        ict.start();
    }

    /**
     * 执行Http请求
     * 
     * @param request 请求数据
     * @param strParaFileName 文件类型的参数名
     * @param strFilePath 文件路径
     * @return 
     * @throws HttpException, IOException 
     */
    public HttpResponse execute(HttpRequest request, String strParaFileName, String strFilePath) throws HttpException, IOException {
        HttpClient httpclient = new HttpClient(connectionManager);

        // 设置连接超时
        int connectionTimeout = defaultConnectionTimeout;
        if (request.getConnectionTimeout() > 0) {
            connectionTimeout = request.getConnectionTimeout();
        }
        httpclient.getHttpConnectionManager().getParams().setConnectionTimeout(connectionTimeout);

        // 设置回应超时
        int soTimeout = defaultSoTimeout;
        if (request.getTimeout() > 0) {
            soTimeout = request.getTimeout();
        }
        httpclient.getHttpConnectionManager().getParams().setSoTimeout(soTimeout);

        // 设置等待ConnectionManager释放connection的时间
        httpclient.getParams().setConnectionManagerTimeout(defaultHttpConnectionManagerTimeout);

        String charset = request.getCharset();
        charset = charset == null ? DEFAULT_CHARSET : charset;
        HttpMethod method = null;

        //get模式且不带上传文件
        if (request.getMethod().equals(HttpRequest.METHOD_GET)) {
            method = new GetMethod(request.getUrl());
            method.getParams().setCredentialCharset(charset);

            // parseNotifyConfig会保证使用GET方法时，request一定使用QueryString
            method.setQueryString(request.getQueryString());
        } else if(strParaFileName.equals("") && strFilePath.equals("")) {
        	//post模式且不带上传文件
            method = new PostMethod(request.getUrl());
            ((PostMethod) method).addParameters(request.getParameters());
            method.addRequestHeader("Content-Type", "application/x-www-form-urlencoded; text/html; charset=" + charset);
        }
        else {
        	//post模式且带上传文件
            method = new PostMethod(request.getUrl());
            List<Part> parts = new ArrayList<Part>();
            for (int i = 0; i < request.getParameters().length; i++) {
            	parts.add(new StringPart(request.getParameters()[i].getName(), request.getParameters()[i].getValue(), charset));
            }
            //增加文件参数，strParaFileName是参数名，使用本地文件
            parts.add(new FilePart(strParaFileName, new FilePartSource(new File(strFilePath))));
            
            // 设置请求体
            ((PostMethod) method).setRequestEntity(new MultipartRequestEntity(parts.toArray(new Part[0]), new HttpMethodParams()));
        }

        // 设置Http Header中的User-Agent属性
        method.addRequestHeader("User-Agent", "Mozilla/4.0");
        HttpResponse response = new HttpResponse();

        try {
            httpclient.executeMethod(method);
            if (request.getResultType().equals(HttpResultType.STRING)) {
                response.setStringResult(method.getResponseBodyAsString());
            } else if (request.getResultType().equals(HttpResultType.BYTES)) {
                response.setByteResult(method.getResponseBody());
            }
            response.setResponseHeaders(method.getResponseHeaders());
        } catch (UnknownHostException ex) {

            return null;
        } catch (IOException ex) {

            return null;
        } catch (Exception ex) {

            return null;
        } finally {
            method.releaseConnection();
        }
        return response;
    }

    /**
     * 将NameValuePairs数组转变为字符串
     * 
     * @param nameValues
     * @return
     */
    protected String toString(NameValuePair[] nameValues) {
        if (nameValues == null || nameValues.length == 0) {
            return "null";
        }

        StringBuffer buffer = new StringBuffer();

        for (int i = 0; i < nameValues.length; i++) {
            NameValuePair nameValue = nameValues[i];

            if (i == 0) {
                buffer.append(nameValue.getName() + "=" + nameValue.getValue());
            } else {
                buffer.append("&" + nameValue.getName() + "=" + nameValue.getValue());
            }
        }

        return buffer.toString();
    }
}



***********************************************************************************

package com.sekorm.core.util.ali.util.httpClient;

import org.apache.commons.httpclient.NameValuePair;

/* *
 *类名：HttpRequest
 *功能：Http请求对象的封装
 *详细：封装Http请求
 *版本：3.3
 *日期：2011-08-17
 *说明：
 *以下代码只是为了方便商户测试而提供的样例代码，商户可以根据自己网站的需要，按照技术文档编写,并非一定要使用该代码。
 *该代码仅供学习和研究支付宝接口使用，只是提供一个参考。
 */

public class HttpRequest {

    /** HTTP GET method */
    public static final String METHOD_GET        = "GET";

    /** HTTP POST method */
    public static final String METHOD_POST       = "POST";

    /**
     * 待请求的url
     */
    private String             url               = null;

    /**
     * 默认的请求方式
     */
    private String             method            = METHOD_POST;

    private int                timeout           = 0;

    private int                connectionTimeout = 0;

    /**
     * Post方式请求时组装好的参数值对
     */
    private NameValuePair[]    parameters        = null;

    /**
     * Get方式请求时对应的参数
     */
    private String             queryString       = null;

    /**
     * 默认的请求编码方式
     */
    private String             charset           = "GBK";

    /**
     * 请求发起方的ip地址
     */
    private String             clientIp;

    /**
     * 请求返回的方式
     */
    private HttpResultType     resultType        = HttpResultType.BYTES;

    public HttpRequest(HttpResultType resultType) {
        super();
        this.resultType = resultType;
    }

    /**
     * @return Returns the clientIp.
     */
    public String getClientIp() {
        return clientIp;
    }

    /**
     * @param clientIp The clientIp to set.
     */
    public void setClientIp(String clientIp) {
        this.clientIp = clientIp;
    }

    public NameValuePair[] getParameters() {
        return parameters;
    }

    public void setParameters(NameValuePair[] parameters) {
        this.parameters = parameters;
    }

    public String getQueryString() {
        return queryString;
    }

    public void setQueryString(String queryString) {
        this.queryString = queryString;
    }

    public String getUrl() {
        return url;
    }

    public void setUrl(String url) {
        this.url = url;
    }

    public String getMethod() {
        return method;
    }

    public void setMethod(String method) {
        this.method = method;
    }

    public int getConnectionTimeout() {
        return connectionTimeout;
    }

    public void setConnectionTimeout(int connectionTimeout) {
        this.connectionTimeout = connectionTimeout;
    }

    public int getTimeout() {
        return timeout;
    }

    public void setTimeout(int timeout) {
        this.timeout = timeout;
    }

    /**
     * @return Returns the charset.
     */
    public String getCharset() {
        return charset;
    }

    /**
     * @param charset The charset to set.
     */
    public void setCharset(String charset) {
        this.charset = charset;
    }

    public HttpResultType getResultType() {
        return resultType;
    }

    public void setResultType(HttpResultType resultType) {
        this.resultType = resultType;
    }

}


***********************************************************************************

package com.sekorm.core.util.ali.util.httpClient;

import java.io.UnsupportedEncodingException;

import org.apache.commons.httpclient.Header;

import com.sekorm.core.util.ali.config.AlipayConfig;

/* *
 *类名：HttpResponse
 *功能：Http返回对象的封装
 *详细：封装Http返回信息
 *版本：3.3
 *日期：2011-08-17
 *说明：
 *以下代码只是为了方便商户测试而提供的样例代码，商户可以根据自己网站的需要，按照技术文档编写,并非一定要使用该代码。
 *该代码仅供学习和研究支付宝接口使用，只是提供一个参考。
 */

public class HttpResponse {

    /**
     * 返回中的Header信息
     */
    private Header[] responseHeaders;

    /**
     * String类型的result
     */
    private String   stringResult;

    /**
     * btye类型的result
     */
    private byte[]   byteResult;

    public Header[] getResponseHeaders() {
        return responseHeaders;
    }

    public void setResponseHeaders(Header[] responseHeaders) {
        this.responseHeaders = responseHeaders;
    }

    public byte[] getByteResult() {
        if (byteResult != null) {
            return byteResult;
        }
        if (stringResult != null) {
            return stringResult.getBytes();
        }
        return null;
    }

    public void setByteResult(byte[] byteResult) {
        this.byteResult = byteResult;
    }

    public String getStringResult() throws UnsupportedEncodingException {
        if (stringResult != null) {
            return stringResult;
        }
        if (byteResult != null) {
            return new String(byteResult, AlipayConfig.input_charset);
        }
        return null;
    }

    public void setStringResult(String stringResult) {
        this.stringResult = stringResult;
    }

}


************************************************************************************


/*
 * Alipay.com Inc.
 * Copyright (c) 2004-2005 All Rights Reserved.
 */
package com.sekorm.core.util.ali.util.httpClient;

/* *
 *类名：HttpResultType
 *功能：表示Http返回的结果字符方式
 *详细：表示Http返回的结果字符方式
 *版本：3.3
 *日期：2012-08-17
 *说明：
 *以下代码只是为了方便商户测试而提供的样例代码，商户可以根据自己网站的需要，按照技术文档编写,并非一定要使用该代码。
 *该代码仅供学习和研究支付宝接口使用，只是提供一个参考。
 */
public enum HttpResultType {
    /**
     * 字符串方式
     */
    STRING,

    /**
     * 字节数组方式
     */
    BYTES
}








====================================================================================



package com.sekorm.core.util.ali.util;

import java.net.URLEncoder;
import java.security.KeyFactory;
import java.security.PrivateKey;
import java.security.spec.PKCS8EncodedKeySpec;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

import com.sekorm.core.util.ali.config.AlipayConfig;
import com.sekorm.core.util.ali.sign.Base64;
import com.sekorm.core.util.ali.sign.RSA;

/**
 * 
 * @describe 专门给APP签名使用
 * 
 * @author bowen_bao
 * @date 2017年5月10日
 */
public class AlipayAppSubmit {
    
	
	public static String getAliPermission(Map<String, String> sParaTemp){
		Map<String, String> sPara = buildRequestPara(sParaTemp);
		String prestr = createLinkString(sPara); 
		return "alipay_sdk=alipay-sdk-java-dynamicVersionNo&"+prestr;
	}
	
	
	/**
	 * 连接参数，并对一级value进行encode  UTF-8
	 * @param params
	 * @return
	 */
	private static String createLinkString(Map<String, String> params) {

        List<String> keys = new ArrayList<String>(params.keySet());
//        Collections.sort(keys);

        String prestr = "";

	        for (int i = 0; i < keys.size(); i++) {
	            String key = keys.get(i);
	            String value = params.get(key);
	            String encodeV=URLEncoder.encode(value);
	            if (i == keys.size() - 1) {//拼接时，不包括最后一个&字符
	                prestr = prestr + key + "=" + encodeV;
	            } else {
	                prestr = prestr + key + "=" + encodeV + "&";
	            }
	        }
        return prestr;
    }
	
	/**
     * 生成签名结果
     * @param sPara 要签名的数组
     * @return 签名结果字符串
     */
	private static String buildRequestMysignForApp(Map<String, String> sPara) {
    	String prestr = createLinkString(sPara); //把数组所有元素，按照“参数=参数值”的模式用“&”字符拼接成字符串
        String mysign = "";
        	if(AlipayConfig.sign_type_APP_ALI.equals("RSA") ){
            	mysign = RSA.sign(prestr, AlipayConfig.private_key_APP_ALI, AlipayConfig.charset, "RSA");
            } else if(AlipayConfig.sign_type_APP_ALI.equals("RSA2")) {
            	mysign = sign(prestr, AlipayConfig.private_key_APP_ALI, AlipayConfig.charset, "RSA2");
            	
        		boolean rs = RSA.verify(prestr, mysign, "MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAnsRQbWhUhY0RZX7yS4SnAh36sdllH3Dj7dEVvSbXYn1gSXo3Z2S3gGQUKQNYLVjCBOIntQnSvkvinK7yP/ukT0zKnGCiZjpKji4mYR5jaJjAPer/vI0FXOZOVCyAR7U9XO+esuN5IjmPKIzcfyzQaHzWfgWjREJXpPu0jOJ9SL3WFcpK5M10SrlMYnxJBlmyfh8oIqW5ouwCgJqUZelkCRXEulerK1H6SIENXG8/OUdx+VMTmAcsh5iGeHOzE/sBVUiGWMT5DfQdwbiGvLG7MDoZ+FbkAlB3hGiXdmyFk8SZ7Q0gCT2efxcpnkBvpEz7NEP7un0ufzU/kgaZebMf0QIDAQAB", "utf-8", "RSA2");
        		
            }
        return mysign;
    }
	
	
	
	
	
	/**
	 * RSA签名
	 * 
	 * @param content
	 *            待签名数据
	 * @param privateKey
	 *            商户私钥
	 * @param input_charset
	 *            编码格式
	 * @param signType
	 *            签名类型      
	 * @return 签名值
	 */
	public static String sign(String content, String privateKey, 
			String input_charset, String signType) {
		try {
			PKCS8EncodedKeySpec priPKCS8 = new PKCS8EncodedKeySpec(
					Base64.decode(privateKey));
			KeyFactory keyf = KeyFactory.getInstance("RSA");
			PrivateKey priKey = keyf.generatePrivate(priPKCS8);

			java.security.Signature signature = null;
	        signature = java.security.Signature.getInstance("SHA256WithRSA");

			signature.initSign(priKey);
//			signature.update(content.getBytes(input_charset));
			signature.update(content.getBytes());
			byte[] signed = signature.sign();

			return Base64.encode(signed);
		} catch (Exception e) {
			e.printStackTrace();
		}

		return null;
	}
	
	
	
    /**
     * 生成要请求给支付宝的参数数组
     * @param sParaTemp 请求前的参数数组
     * @return 要请求的参数数组
     */
    private static Map<String, String> buildRequestPara(Map<String, String> sParaTemp) {
        //除去数组中的空值和签名参数
        Map<String, String> sPara = AlipayCore.paraFilter(sParaTemp);
        //生成签名结果
        String mysign = buildRequestMysignForApp(sPara);

        //签名结果与签名方式加入请求提交参数组中
        sPara.put("sign", mysign);
        sPara.put("sign_type", AlipayConfig.sign_type_APP_ALI);

        return sPara;
    }

    
    
}



***************************************************************************************


package com.sekorm.core.util.ali.util;

import java.io.File;
import java.io.FileWriter;
import java.io.IOException;
import java.util.ArrayList;
import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import org.apache.commons.codec.digest.DigestUtils;
import org.apache.commons.httpclient.methods.multipart.FilePartSource;
import org.apache.commons.httpclient.methods.multipart.PartSource;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.sekorm.core.util.ali.config.AlipayConfig;

/* *
 *类名：AlipayFunction
 *功能：支付宝接口公用函数类
 *详细：该类是请求、通知返回两个文件所调用的公用函数核心处理文件，不需要修改
 *版本：3.3
 *日期：2012-08-14
 *说明：
 *以下代码只是为了方便商户测试而提供的样例代码，商户可以根据自己网站的需要，按照技术文档编写,并非一定要使用该代码。
 *该代码仅供学习和研究支付宝接口使用，只是提供一个参考。
 */

public class AlipayCore {
	
	private static final Logger log = LoggerFactory.getLogger(AlipayCore.class);

    /** 
     * 除去数组中的空值和签名参数
     * @param sArray 签名参数组
     * @return 去掉空值与签名参数后的新签名参数组
     */
    public static Map<String, String> paraFilter(Map<String, String> sArray) {

        Map<String, String> result = new HashMap<String, String>();

        if (sArray == null || sArray.size() <= 0) {
            return result;
        }

        for (String key : sArray.keySet()) {
            String value = sArray.get(key);
            if (value == null || value.equals("") || key.equalsIgnoreCase("sign")
                || key.equalsIgnoreCase("sign_type")) {
                continue;
            }
            result.put(key, value);
        }

        return result;
    }

    /** 
     * 把数组所有元素排序，并按照“参数=参数值”的模式用“&”字符拼接成字符串
     * @param params 需要排序并参与字符拼接的参数组
     * @return 拼接后字符串
     */
    public static String createLinkString(Map<String, String> params) {

        List<String> keys = new ArrayList<String>(params.keySet());
        Collections.sort(keys);

        String prestr = "";

        for (int i = 0; i < keys.size(); i++) {
            String key = keys.get(i);
            String value = params.get(key);

            if (i == keys.size() - 1) {//拼接时，不包括最后一个&字符
                prestr = prestr + key + "=" + value;
            } else {
                prestr = prestr + key + "=" + value + "&";
            }
        }

        return prestr;
    }

    /** 
     * 写日志，方便测试（看网站需求，也可以改成把记录存入数据库）
     * @param sWord 要写入日志里的文本内容
     */
    public static void logResult(String sWord) {
//        FileWriter writer = null;
        try {
//            writer = new FileWriter(AlipayConfig.log_path + "alipay_log_test.log");
//            writer.append(sWord);
        	log.info(sWord);
        } catch (Exception e) {
            e.printStackTrace();
        } 
//        finally {
//            if (writer != null) {
//                try {
//                    writer.close();
//                } catch (IOException e) {
//                    e.printStackTrace();
//                }
//            }
//        }
    }

    /** 
     * 生成文件摘要
     * @param strFilePath 文件路径
     * @param file_digest_type 摘要算法
     * @return 文件摘要结果
     */
    public static String getAbstract(String strFilePath, String file_digest_type) throws IOException {
        PartSource file = new FilePartSource(new File(strFilePath));
    	if(file_digest_type.equals("MD5")){
    		return DigestUtils.md5Hex(file.createInputStream());
    	}
    	else if(file_digest_type.equals("SHA")) {
    		return DigestUtils.sha256Hex(file.createInputStream());
    	}
    	else {
    		return "";
    	}
    }
}




***********************************************************************************************


package com.sekorm.core.util.ali.util;

import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.net.HttpURLConnection;
import java.net.URL;
import java.util.Map;

import com.sekorm.core.util.ali.config.AlipayConfig;
import com.sekorm.core.util.ali.sign.RSA;

/* *
 *类名：AlipayNotify
 *功能：支付宝通知处理类
 *详细：处理支付宝各接口通知返回
 *版本：3.3
 *日期：2012-08-17
 *说明：
 *以下代码只是为了方便商户测试而提供的样例代码，商户可以根据自己网站的需要，按照技术文档编写,并非一定要使用该代码。
 *该代码仅供学习和研究支付宝接口使用，只是提供一个参考

 *************************注意*************************
 *调试通知返回时，可查看或改写log日志的写入TXT里的数据，来检查通知返回是否正常
 */
public class AlipayNotify {

    /**
     * 支付宝消息验证地址
     */
    private static final String HTTPS_VERIFY_URL = "https://mapi.alipay.com/gateway.do?service=notify_verify&";

    /**
     * 验证消息是否是支付宝发出的合法消息
     * @param params 通知返回来的参数数组
     * @return 验证结果
     */
    public static boolean verify(Map<String, String> params) {

        //判断responsetTxt是否为true，isSign是否为true
        //responsetTxt的结果不是true，与服务器设置问题、合作身份者ID、notify_id一分钟失效有关
        //isSign不是true，与安全校验码、请求时的参数格式（如：带自定义参数等）、编码格式有关
    	String responseTxt = "false";
		if(params.get("notify_id") != null) {
			String notify_id = params.get("notify_id");
			responseTxt = verifyResponse(notify_id);
		}
	    String sign = "";
	    if(params.get("sign") != null) {sign = params.get("sign");}
	    boolean isSign = getSignVeryfy(params, sign);

        //写日志记录（若要调试，请取消下面两行注释）
        String sWord = "responseTxt=" + responseTxt + "\n isSign=" + isSign + "\n 返回回来的参数：" + AlipayCore.createLinkString(params);
	    AlipayCore.logResult(sWord);

        if (isSign && responseTxt.equals("true")) {
            return true;
        } else {
            return false;
        }
    }

    /**
     * 根据反馈回来的信息，生成签名结果
     * @param Params 通知返回来的参数数组
     * @param sign 比对的签名结果
     * @return 生成的签名结果
     */
	private static boolean getSignVeryfy(Map<String, String> Params, String sign) {
    	//过滤空值、sign与sign_type参数
    	Map<String, String> sParaNew = AlipayCore.paraFilter(Params);
        //获取待签名字符串
        String preSignStr = AlipayCore.createLinkString(sParaNew);
        //获得签名验证结果
        boolean isSign = false;
        if(AlipayConfig.sign_type.equals("RSA")){
        	isSign = RSA.verify(preSignStr, sign, AlipayConfig.alipay_public_key, AlipayConfig.input_charset, "RSA");
        } else if(AlipayConfig.sign_type.equals("RSA2")) {
        	isSign = RSA.verify(preSignStr, sign, AlipayConfig.alipay_public_key, AlipayConfig.input_charset, "RSA2");
        }
        return isSign;
    }

    /**
    * 获取远程服务器ATN结果,验证返回URL
    * @param notify_id 通知校验ID
    * @return 服务器ATN结果
    * 验证结果集：
    * invalid命令参数不对 出现这个错误，请检测返回处理中partner和key是否为空 
    * true 返回正确信息
    * false 请检查防火墙或者是服务器阻止端口问题以及验证时间是否超过一分钟
    */
    private static String verifyResponse(String notify_id) {
        //获取远程服务器ATN结果，验证是否是支付宝服务器发来的请求

        String partner = AlipayConfig.partner;
        String veryfy_url = HTTPS_VERIFY_URL + "partner=" + partner + "&notify_id=" + notify_id;

        return checkUrl(veryfy_url);
    }

    /**
    * 获取远程服务器ATN结果
    * @param urlvalue 指定URL路径地址
    * @return 服务器ATN结果
    * 验证结果集：
    * invalid命令参数不对 出现这个错误，请检测返回处理中partner和key是否为空 
    * true 返回正确信息
    * false 请检查防火墙或者是服务器阻止端口问题以及验证时间是否超过一分钟
    */
    private static String checkUrl(String urlvalue) {
        String inputLine = "";

        try {
            URL url = new URL(urlvalue);
            HttpURLConnection urlConnection = (HttpURLConnection) url.openConnection();
            BufferedReader in = new BufferedReader(new InputStreamReader(urlConnection
                .getInputStream()));
            inputLine = in.readLine().toString();
        } catch (Exception e) {
            e.printStackTrace();
            inputLine = "";
        }

        return inputLine;
    }
}




*****************************************************************************************


package com.sekorm.core.util.ali.util;

import java.io.IOException;
import java.net.MalformedURLException;
import java.net.URL;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

import org.dom4j.Document;
import org.dom4j.DocumentException;
import org.dom4j.Node;
import org.dom4j.io.SAXReader;

import com.sekorm.core.util.ali.config.AlipayConfig;
import com.sekorm.core.util.ali.sign.RSA;

/* *
 *类名：AlipaySubmit
 *功能：支付宝各接口请求提交类
 *详细：构造支付宝各接口表单HTML文本，获取远程HTTP数据
 *版本：3.3
 *日期：2012-08-13
 *说明：
 *以下代码只是为了方便商户测试而提供的样例代码，商户可以根据自己网站的需要，按照技术文档编写,并非一定要使用该代码。
 *该代码仅供学习和研究支付宝接口使用，只是提供一个参考。
 */

public class AlipaySubmit {
    
    /**
     * 支付宝提供给商户的服务接入网关URL(新)
     */
    private static final String ALIPAY_GATEWAY_NEW = AlipayConfig.alipay_gateway_new;
	
    /**
     * 生成签名结果
     * @param sPara 要签名的数组
     * @return 签名结果字符串
     */
	public static String buildRequestMysign(Map<String, String> sPara) {
    	String prestr = AlipayCore.createLinkString(sPara); //把数组所有元素，按照“参数=参数值”的模式用“&”字符拼接成字符串
        String mysign = "";
        if(AlipayConfig.sign_type.equals("RSA") ){
        	mysign = RSA.sign(prestr, AlipayConfig.private_key, AlipayConfig.input_charset, "RSA");
        } else if(AlipayConfig.sign_type.equals("RSA2")) {
        	mysign = RSA.sign(prestr, AlipayConfig.private_key, AlipayConfig.input_charset, "RSA2");
        }
        return mysign;
    }
	
    /**
     * 生成要请求给支付宝的参数数组
     * @param sParaTemp 请求前的参数数组
     * @return 要请求的参数数组
     */
    private static Map<String, String> buildRequestPara(Map<String, String> sParaTemp) {
        //除去数组中的空值和签名参数
        Map<String, String> sPara = AlipayCore.paraFilter(sParaTemp);
        //生成签名结果
        String mysign = buildRequestMysign(sPara);

        //签名结果与签名方式加入请求提交参数组中
        sPara.put("sign", mysign);
        sPara.put("sign_type", AlipayConfig.sign_type);

        return sPara;
    }

    /**
     * 建立请求，以表单HTML形式构造（默认）
     * @param sParaTemp 请求参数数组
     * @param strMethod 提交方式。两个值可选：post、get
     * @param strButtonName 确认按钮显示文字
     * @return 提交表单HTML文本
     */
    public static String buildRequest(Map<String, String> sParaTemp, String strMethod, String strButtonName) {
        //待请求参数数组
        Map<String, String> sPara = buildRequestPara(sParaTemp);
        List<String> keys = new ArrayList<String>(sPara.keySet());

        StringBuffer sbHtml = new StringBuffer("<html>"
        		+ "<head><meta http-equiv=\"Content-Type\" content=\"text/html; charset=UTF-8\">"
        		+ "<title>支付宝即时到账交易接口</title>"
        		+ "</head><body>");

        sbHtml.append("<form id=\"alipaysubmit\" name=\"alipaysubmit\" action=\"" + ALIPAY_GATEWAY_NEW
                      + "\" method=\"" + strMethod + "\">");

        for (int i = 0; i < keys.size(); i++) {
            String name = (String) keys.get(i);
            String value = (String) sPara.get(name);

            sbHtml.append("<input type=\"hidden\" name=\"" + name + "\" value=\"" + value + "\"/>");
        }

        //submit按钮控件请不要含有name属性
        sbHtml.append("<input type=\"submit\" value=\"" + strButtonName + "\" style=\"display:none;\"></form></body>");
        sbHtml.append("<script>document.forms['alipaysubmit'].submit();</script></html>");
        return sbHtml.toString();
    }
    
 
    
    /**
     * 用于防钓鱼，调用接口query_timestamp来获取时间戳的处理函数
     * 注意：远程解析XML出错，与服务器是否支持SSL等配置有关
     * @return 时间戳字符串
     * @throws IOException
     * @throws DocumentException
     * @throws MalformedURLException
     */
	@SuppressWarnings("unchecked")
	public static String query_timestamp() throws MalformedURLException,
                                                        DocumentException, IOException {
        //构造访问query_timestamp接口的URL串
        String strUrl = ALIPAY_GATEWAY_NEW + "service=query_timestamp&partner=" + AlipayConfig.partner + "&_input_charset" +AlipayConfig.input_charset;
        StringBuffer result = new StringBuffer();

        SAXReader reader = new SAXReader();
        Document doc = reader.read(new URL(strUrl).openStream());

        List<Node> nodeList = doc.selectNodes("//alipay/*");

        for (Node node : nodeList) {
            // 截取部分不需要解析的信息
            if (node.getName().equals("is_success") && node.getText().equals("T")) {
                // 判断是否有成功标示
                List<Node> nodeList1 = doc.selectNodes("//response/timestamp/*");
                for (Node node1 : nodeList1) {
                    result.append(node1.getText());
                }
            }
        }
        return result.toString();
    }
}




```

#### wx

##### request


```

package com.sekorm.core.util.wx.request;

import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;

import com.sekorm.core.util.wx.util.Configure;
import com.sekorm.core.util.wx.util.RandomStringGenerator;
import com.sekorm.core.util.wx.util.Signature;


/**
 * User: rizenguo
 * Date: 2014/10/25
 * Time: 16:48
 */
public class DownloadBillReqData {
    //每个字段具体的意思请查看API文档
    private String appid = "";
    private String mch_id = "";
    private String device_info = "";
    private String nonce_str = "";
    private String sign = "";
    private String bill_date = "";
    private String bill_type = "";
    private String sdk_version = "";

    /**
     * 请求对账单下载服务
     * @param deviceInfo 商户自己定义的扫码支付终端设备号，方便追溯这笔交易发生在哪台终端设备上
     * @param billDate 下载对账单的日期，格式：yyyyMMdd 例如：20140603
     * @param billType 账单类型
     *                 ALL，返回当日所有订单信息，默认值
    SUCCESS，返回当日成功支付的订单
    REFUND，返回当日退款订单
    REVOKED，已撤销的订单
     */
    public DownloadBillReqData(String deviceInfo,String billDate,String billType){

        setSdk_version(Configure.getSdkVersion());

        //微信分配的公众号ID（开通公众号之后可以获取到）
        setAppid(Configure.appID);

        //微信支付分配的商户号ID（开通公众号的微信支付功能之后可以获取到）
        setMch_id(Configure.mchID);

        //商户自己定义的扫码支付终端设备号，方便追溯这笔交易发生在哪台终端设备上
        setDevice_info(deviceInfo);

        setBill_date(billDate);

        setBill_type(billType);


        //随机字符串，不长于32 位
        setNonce_str(RandomStringGenerator.getRandomStringByLength(32));

        //根据API给的签名规则进行签名
        String sign = Signature.getSign(toMap());
        setSign(sign);//把签名数据设置到Sign这个属性中


    }

    public String getAppid() {
        return appid;
    }

    public void setAppid(String appid) {
        this.appid = appid;
    }

    public String getMch_id() {
        return mch_id;
    }

    public void setMch_id(String mch_id) {
        this.mch_id = mch_id;
    }

    public String getDevice_info() {
        return device_info;
    }

    public void setDevice_info(String device_info) {
        this.device_info = device_info;
    }

    public String getNonce_str() {
        return nonce_str;
    }

    public void setNonce_str(String nonce_str) {
        this.nonce_str = nonce_str;
    }

    public String getSign() {
        return sign;
    }

    public void setSign(String sign) {
        this.sign = sign;
    }

    public String getBill_date() {
        return bill_date;
    }

    public void setBill_date(String bill_date) {
        this.bill_date = bill_date;
    }

    public String getBill_type() {
        return bill_type;
    }

    public void setBill_type(String bill_type) {
        this.bill_type = bill_type;
    }

    public String getSdk_version(){
        return sdk_version;
    }

    public void setSdk_version(String sdk_version) {
        this.sdk_version = sdk_version;
    }

    public Map<String,Object> toMap(){
        Map<String,Object> map = new HashMap<String, Object>();
        Field[] fields = this.getClass().getDeclaredFields();
        for (Field field : fields) {
            Object obj;
            try {
                obj = field.get(this);
                if(obj!=null){
                    map.put(field.getName(), obj);
                }
            } catch (IllegalArgumentException e) {
                e.printStackTrace();
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            }
        }
        return map;
    }

}



******************************************************************************

package com.sekorm.core.util.wx.request;


import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;

import com.sekorm.core.util.wx.util.Configure;
import com.sekorm.core.util.wx.util.RandomStringGenerator;
import com.sekorm.core.util.wx.util.Signature;

/**
 * User: rizenguo
 * Date: 2014/10/25
 * Time: 13:54
 */
public class ScanPayQueryReqData {

    //每个字段具体的意思请查看API文档
    private String appid = "";
    private String mch_id = "";
    private String transaction_id = "";
    private String out_trade_no = "";
    private String nonce_str = "";
    private String sign = "";
    private String sdk_version = "";

    /**
     * 请求支付查询服务
     * @param transactionID 是微信系统为每一笔支付交易分配的订单号，通过这个订单号可以标识这笔交易，它由支付订单API支付成功时返回的数据里面获取到。建议优先使用
     * @param outTradeNo 商户系统内部的订单号,transaction_id 、out_trade_no 二选一，如果同时存在优先级：transaction_id>out_trade_no
     * @return API返回的XML数据
     * @throws Exception
     */
    public ScanPayQueryReqData(String transactionID, String outTradeNo){

        //--------------------------------------------------------------------
        //以下是测试数据，请商户按照自己的实际情况填写具体的值进去
        //--------------------------------------------------------------------

        setSdk_version(Configure.getSdkVersion());

        //微信分配的公众号ID（开通公众号之后可以获取到）
        setAppid(Configure.appID);

        //微信支付分配的商户号ID（开通公众号的微信支付功能之后可以获取到）
        setMch_id(Configure.mchID);

        //transaction_id是微信系统为每一笔支付交易分配的订单号，通过这个订单号可以标识这笔交易，它由支付订单API支付成功时返回的数据里面获取到。
        setTransaction_id(transactionID);

        //商户系统自己生成的唯一的订单号
        setOut_trade_no(outTradeNo);

        //随机字符串，不长于32 位
        setNonce_str(RandomStringGenerator.getRandomStringByLength(32));

        //根据API给的签名规则进行签名
        String sign = Signature.getSign(toMap());
        setSign(sign);//把签名数据设置到Sign这个属性中


    }

    public String getAppid() {
        return appid;
    }

    public void setAppid(String appid) {
        this.appid = appid;
    }

    public String getMch_id() {
        return mch_id;
    }

    public void setMch_id(String mch_id) {
        this.mch_id = mch_id;
    }

    public String getTransaction_id() {
        return transaction_id;
    }

    public void setTransaction_id(String transaction_id) {
        this.transaction_id = transaction_id;
    }

    public String getOut_trade_no() {
        return out_trade_no;
    }

    public void setOut_trade_no(String out_trade_no) {
        this.out_trade_no = out_trade_no;
    }

    public String getNonce_str() {
        return nonce_str;
    }

    public void setNonce_str(String nonce_str) {
        this.nonce_str = nonce_str;
    }

    public String getSign() {
        return sign;
    }

    public void setSign(String sign) {
        this.sign = sign;
    }

    public String getSdk_version(){
        return sdk_version;
    }

    public void setSdk_version(String sdk_version) {
        this.sdk_version = sdk_version;
    }

    public Map<String,Object> toMap(){
        Map<String,Object> map = new HashMap<String, Object>();
        Field[] fields = this.getClass().getDeclaredFields();
        for (Field field : fields) {
            Object obj;
            try {
                obj = field.get(this);
                if(obj!=null){
                    map.put(field.getName(), obj);
                }
            } catch (IllegalArgumentException e) {
                e.printStackTrace();
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            }
        }
        return map;
    }
}



**********************************************************************


package com.sekorm.core.util.wx.request;

import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;

import com.sekorm.core.util.wx.util.Configure;
import com.sekorm.core.util.wx.util.RandomStringGenerator;


public class ScanPayReqData {
	/*
	字段名	变量名	必填	类型	示例值	描述
	公众账号ID	appid	是	String(32)	wxd678efh567hg6787	微信支付分配的公众账号ID（企业号corpid即为此appId）
	商户号	mch_id	是	String(32)	1230000109	微信支付分配的商户号
	设备号	device_info	否	String(32)	013467007045764	自定义参数，可以为终端设备号(门店号或收银设备ID)，PC网页或公众号内支付可以传"WEB"
	随机字符串	nonce_str	是	String(32)	5K8264ILTKCH16CQ2502SI8ZNMTM67VS	随机字符串，长度要求在32位以内。推荐随机数生成算法
	签名	sign	是	String(32)	C380BEC2BFD727A4B6845133519F3AD6	通过签名算法计算得出的签名值，详见签名生成算法
	签名类型	sign_type	否	String(32)	HMAC-SHA256	签名类型，默认为MD5，支持HMAC-SHA256和MD5。
	商品描述	body	是	String(128)	腾讯充值中心-QQ会员充值	
	商品简单描述，该字段请按照规范传递，具体请见参数规定
	商品详情	detail	否	String(6000)	 	单品优惠字段(暂未上线)
	附加数据	attach	否	String(127)	深圳分店	附加数据，在查询API和支付通知中原样返回，可作为自定义参数使用。
	商户订单号	out_trade_no	是	String(32)	20150806125346	商户系统内部订单号，要求32个字符内、且在同一个商户号下唯一。 详见商户订单号
	标价币种	fee_type	否	String(16)	CNY	符合ISO 4217标准的三位字母代码，默认人民币：CNY，详细列表请参见货币类型
	标价金额	total_fee	是	Int	88	订单总金额，单位为分，详见支付金额
	终端IP	spbill_create_ip	是	String(16)	123.12.12.123	APP和网页支付提交用户端ip，Native支付填调用微信支付API的机器IP。
	交易起始时间	time_start	否	String(14)	20091225091010	订单生成时间，格式为yyyyMMddHHmmss，如2009年12月25日9点10分10秒表示为20091225091010。其他详见时间规则
	交易结束时间	time_expire	否	String(14)	20091227091010	
	订单失效时间，格式为yyyyMMddHHmmss，如2009年12月27日9点10分10秒表示为20091227091010。其他详见时间规则
	注意：最短失效时间间隔必须大于5分钟
	商品标记	goods_tag	否	String(32)	WXG	商品标记，使用代金券或立减优惠功能时需要的参数，说明详见代金券或立减优惠
	通知地址	notify_url	是	String(256)	http://www.weixin.qq.com/wxpay/pay.php	异步接收微信支付结果通知的回调地址，通知url必须为外网可访问的url，不能携带参数。
	交易类型	trade_type	是	String(16)	JSAPI	取值如下：JSAPI，NATIVE，APP等，说明详见参数规定
	商品ID	product_id	否	String(32)	12235413214070356458058	trade_type=NATIVE时（即扫码支付），此参数必传。此参数为二维码中包含的商品ID，商户自行定义。
	指定支付方式	limit_pay	否	String(32)	no_credit	上传此参数no_credit--可限制用户不能使用信用卡支付
	用户标识	openid	否	String(128)	oUpF8uMuAJO_M2pxb1Q9zNjWeS6o	trade_type=JSAPI时（即公众号支付），此参数必传，此参数为微信用户在商户对应appid下的唯一标识。openid如何获取，可参考【获取openid】。企业号请使用【企业号OAuth2.0接口】获取企业号内成员userid，再调用【企业号userid转openid接口】进行转换
	*/
	String appid = "";				//公众账号ID
	String mch_id = "";				//商户号
	String device_info = "WEB";		//设备号：待定是否加上
	String nonce_str = "";			//随机字符串
	String sign = "";				//签名
	String body = "";				//商品描述
	String out_trade_no = "";		//商户订单号
	Integer total_fee;				//标价金额
	String spbill_create_ip = "";	//终端IP
	String notify_url = "";			//通知地址
	String trade_type = "";			//交易类型
	
	/**
	 * 请求统一下单服务
	 * @param deviceInfo
	 */
	public ScanPayReqData() {
		//微信分配的公众号ID（开通公众号之后可以获取到）
        setAppid(Configure.appID);

        //微信支付分配的商户号ID（开通公众号的微信支付功能之后可以获取到）
        setMch_id(Configure.mchID);

        //随机字符串，不长于32 位
        setNonce_str(RandomStringGenerator.getRandomStringByLength(32));

        //根据API给的签名规则进行签名
//        String sign = Signature.getSign(toMap());
//        setSign(sign);//把签名数据设置到Sign这个属性中
	}
	
	public String getAppid() {
		return appid;
	}
	public void setAppid(String appid) {
		this.appid = appid;
	}
	public String getMch_id() {
		return mch_id;
	}
	public void setMch_id(String mch_id) {
		this.mch_id = mch_id;
	}
	public String getDevice_info() {
		return device_info;
	}
	public void setDevice_info(String device_info) {
		this.device_info = device_info;
	}
	public String getNonce_str() {
		return nonce_str;
	}
	public void setNonce_str(String nonce_str) {
		this.nonce_str = nonce_str;
	}
	public String getSign() {
		return sign;
	}
	public void setSign(String sign) {
		this.sign = sign;
	}
	public String getBody() {
		return body;
	}
	public void setBody(String body) {
		this.body = body;
	}
	public String getOut_trade_no() {
		return out_trade_no;
	}
	public void setOut_trade_no(String out_trade_no) {
		this.out_trade_no = out_trade_no;
	}
	public Integer getTotal_fee() {
		return total_fee;
	}
	public void setTotal_fee(Integer total_fee) {
		this.total_fee = total_fee;
	}
	public String getSpbill_create_ip() {
		return spbill_create_ip;
	}
	public void setSpbill_create_ip(String spbill_create_ip) {
		this.spbill_create_ip = spbill_create_ip;
	}
	public String getNotify_url() {
		return notify_url;
	}
	public void setNotify_url(String notify_url) {
		this.notify_url = notify_url;
	}
	public String getTrade_type() {
		return trade_type;
	}
	public void setTrade_type(String trade_type) {
		this.trade_type = trade_type;
	}
	
	public Map<String,Object> toMap(){
        Map<String,Object> map = new HashMap<String, Object>();
        Field[] fields = this.getClass().getDeclaredFields();
        for (Field field : fields) {
            Object obj;
            try {
                obj = field.get(this);
                if(obj!=null){
                    map.put(field.getName(), obj);
                }
            } catch (IllegalArgumentException e) {
                e.printStackTrace();
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            }
        }
        return map;
    }
}



```


##### response

```

package com.sekorm.core.util.wx.response;

/**
 * User: rizenguo
 * Date: 2014/10/25
 * Time: 16:48
 */
public class DownloadBillResData {

    //协议层
    private String return_code = "";
    private String return_msg = "";

    public String getReturn_code() {
        return return_code;
    }

    public void setReturn_code(String return_code) {
        this.return_code = return_code;
    }

    public String getReturn_msg() {
        return return_msg;
    }

    public void setReturn_msg(String return_msg) {
        this.return_msg = return_msg;
    }
}



**************************************************************


package com.sekorm.core.util.wx.response;

public class NotifyReturnData {

	/*
返回状态码	return_code	是	String(16)	SUCCESS	SUCCESS/FAIL SUCCESS表示商户接收通知成功并校验成功
返回信息	return_msg	否	String(128)	OK		返回信息，如非空，为错误原因：签名失败参数格式校验错误
	 */
	private String return_code = "";	//返回状态码
    private String return_msg = "";		//返回信息
	
    public NotifyReturnData(String return_code, String return_msg) {
    	this.return_code = return_code;
    	this.return_msg = return_msg;
    }

	public String getReturn_code() {
		return return_code;
	}

	public void setReturn_code(String return_code) {
		this.return_code = return_code;
	}

	public String getReturn_msg() {
		return return_msg;
	}

	public void setReturn_msg(String return_msg) {
		this.return_msg = return_msg;
	}
}



**************************************************************


package com.sekorm.core.util.wx.response;

/**
 * User: rizenguo
 * Date: 2014/10/25
 * Time: 13:54
 */
public class ScanPayQueryResData {

    //协议层
    private String return_code = "";
    private String return_msg = "";

    //协议返回的具体数据（以下字段在return_code 为SUCCESS 的时候有返回）
    private String appid = "";
    private String mch_id = "";
    private String sub_mch_id = "";//新增
    private String nonce_str = "";
    private String sign = "";
    private String result_code = "";
    private String err_code = "";
    private String err_code_des = "";

    //以下字段在return_code 和result_code 都为SUCCESS 的时候有返回
    private String trade_state = "";

    //trade_state的几种可能取值：
    //    SUCCESS--支付成功
    //    REFUND--转入退款
    //    NOTPAY--未支付
    //    CLOSED--已关闭
    //    REVOKED--已撤销
    //    USERPAYING--用户支付中
    //    NOPAY--未支付(确认支付超时)
    //    PAYERROR--支付失败(其他原因，
    //            如银行返回失败)

    //以下字段在trade_state 为SUCCESS 或者REFUND 的时候有返回
    private String device_info = "";
    private String openid = "";
    private String is_subscribe = "";
    private String trade_type = "";
    private String bank_type = "";
    private String total_fee = "";
    private String coupon_fee = "";
    private String fee_type = "";
    private String cash_fee = "";
    private String cash_fee_type = "";
    private String transaction_id = "";
    private String out_trade_no = "";
    private String attach = "";
    private String time_end = "";

    public String getReturn_code() {
        return return_code;
    }

    public void setReturn_code(String return_code) {
        this.return_code = return_code;
    }

    public String getReturn_msg() {
        return return_msg;
    }

    public void setReturn_msg(String return_msg) {
        this.return_msg = return_msg;
    }

    public String getAppid() {
        return appid;
    }

    public void setAppid(String appid) {
        this.appid = appid;
    }

    public String getMch_id() {
        return mch_id;
    }

    public void setMch_id(String mch_id) {
        this.mch_id = mch_id;
    }

    public String getSub_mch_id() {
        return sub_mch_id;
    }

    public void setSub_mch_id(String sub_mch_id) {
        this.sub_mch_id = sub_mch_id;
    }

    public String getNonce_str() {
        return nonce_str;
    }

    public void setNonce_str(String nonce_str) {
        this.nonce_str = nonce_str;
    }

    public String getSign() {
        return sign;
    }

    public void setSign(String sign) {
        this.sign = sign;
    }

    public String getResult_code() {
        return result_code;
    }

    public void setResult_code(String result_code) {
        this.result_code = result_code;
    }

    public String getErr_code() {
        return err_code;
    }

    public void setErr_code(String err_code) {
        this.err_code = err_code;
    }

    public String getErr_code_des() {
        return err_code_des;
    }

    public void setErr_code_des(String err_code_des) {
        this.err_code_des = err_code_des;
    }

    public String getTrade_state() {
        return trade_state;
    }

    public void setTrade_state(String trade_state) {
        this.trade_state = trade_state;
    }

    public String getDevice_info() {
        return device_info;
    }

    public void setDevice_info(String device_info) {
        this.device_info = device_info;
    }

    public String getOpenid() {
        return openid;
    }

    public void setOpenid(String openid) {
        this.openid = openid;
    }

    public String getIs_subscribe() {
        return is_subscribe;
    }

    public void setIs_subscribe(String is_subscribe) {
        this.is_subscribe = is_subscribe;
    }

    public String getTrade_type() {
        return trade_type;
    }

    public void setTrade_type(String trade_type) {
        this.trade_type = trade_type;
    }

    public String getBank_type() {
        return bank_type;
    }

    public void setBank_type(String bank_type) {
        this.bank_type = bank_type;
    }

    public String getTotal_fee() {
        return total_fee;
    }

    public void setTotal_fee(String total_fee) {
        this.total_fee = total_fee;
    }

    public String getCoupon_fee() {
        return coupon_fee;
    }

    public void setCoupon_fee(String coupon_fee) {
        this.coupon_fee = coupon_fee;
    }

    public String getFee_type() {
        return fee_type;
    }

    public void setFee_type(String fee_type) {
        this.fee_type = fee_type;
    }

    public String getTransaction_id() {
        if(transaction_id != null) {
            return transaction_id;
        }else{
            return "";
        }
    }

    public void setTransaction_id(String transaction_id) {
        this.transaction_id = transaction_id;
    }

    public String getOut_trade_no() {
        return out_trade_no;
    }

    public void setOut_trade_no(String out_trade_no) {
        this.out_trade_no = out_trade_no;
    }

    public String getAttach() {
        return attach;
    }

    public void setAttach(String attach) {
        this.attach = attach;
    }

    public String getTime_end() {
        return time_end;
    }

    public void setTime_end(String time_end) {
        this.time_end = time_end;
    }

    public String getCash_fee() {
        return cash_fee;
    }

    public void setCash_fee(String cash_fee) {
        this.cash_fee = cash_fee;
    }

    public String getCash_fee_type() {
        return cash_fee_type;
    }

    public void setCash_fee_type(String cash_fee_type) {
        this.cash_fee_type = cash_fee_type;
    }
}




*******************************************************************


package com.sekorm.core.util.wx.response;

public class ScanPayResData {

	/*
返回状态码	return_code	是	String(16)	SUCCESS	SUCCESS/FAIL
此字段是通信标识，非交易标识，交易是否成功需要查看result_code来判断
返回信息	return_msg	否	String(128)	签名失败	
返回信息，如非空，为错误原因，签名失败、参数格式校验错误

以下字段在return_code为SUCCESS的时候有返回

字段名	变量名	必填	类型	示例值	描述
公众账号ID	appid	是	String(32)	wx8888888888888888	调用接口提交的公众账号ID
商户号	mch_id	是	String(32)	1900000109	调用接口提交的商户号
设备号	device_info	否	String(32)	013467007045764	自定义参数，可以为请求支付的终端设备号等
随机字符串	nonce_str	是	String(32)	5K8264ILTKCH16CQ2502SI8ZNMTM67VS	微信返回的随机字符串
签名	sign	是	String(32)	C380BEC2BFD727A4B6845133519F3AD6	微信返回的签名值，详见签名算法
业务结果	result_code	是	String(16)	SUCCESS	SUCCESS/FAIL
错误代码	err_code	否	String(32)	SYSTEMERROR	详细参见下文错误列表
错误代码描述	err_code_des	否	String(128)	系统错误	错误信息描述

以下字段在return_code 和result_code都为SUCCESS的时候有返回

字段名	变量名	必填	类型	示例值	描述
交易类型	trade_type	是	String(16)	JSAPI	交易类型，取值为：JSAPI，NATIVE，APP等，说明详见参数规定
预支付交易会话标识	prepay_id	是	String(64)	wx201410272009395522657a690389285100	微信生成的预支付会话标识，用于后续接口调用中使用，该值有效期为2小时
二维码链接	code_url	否	String(64)	URl：weixin：//wxpay/s/An4baqw	trade_type为NATIVE时有返回，用于生成二维码，展示给用户进行扫码支付
	 */
	
    //协议层
    private String return_code = "";
    private String return_msg = "";

    //协议返回的具体数据（以下字段在return_code 为SUCCESS 的时候有返回）
    private String appid = "";
    private String mch_id = "";
    private String nonce_str = "";
    private String sign = "";
    private String result_code = "";
    private String err_code = "";
    private String err_code_des = "";

    //以下字段在return_code 和result_code 都为SUCCESS 的时候有返回
    private String trade_type = "";
    private String prepay_id = "";
    private String code_url = "";
    
    
    private String rs;
    
	public String getReturn_code() {
		return return_code;
	}
	public void setReturn_code(String return_code) {
		this.return_code = return_code;
	}
	public String getReturn_msg() {
		return return_msg;
	}
	public void setReturn_msg(String return_msg) {
		this.return_msg = return_msg;
	}
	public String getAppid() {
		return appid;
	}
	public void setAppid(String appid) {
		this.appid = appid;
	}
	public String getMch_id() {
		return mch_id;
	}
	public void setMch_id(String mch_id) {
		this.mch_id = mch_id;
	}
	public String getNonce_str() {
		return nonce_str;
	}
	public void setNonce_str(String nonce_str) {
		this.nonce_str = nonce_str;
	}
	public String getSign() {
		return sign;
	}
	public void setSign(String sign) {
		this.sign = sign;
	}
	public String getResult_code() {
		return result_code;
	}
	public void setResult_code(String result_code) {
		this.result_code = result_code;
	}
	public String getErr_code() {
		return err_code;
	}
	public void setErr_code(String err_code) {
		this.err_code = err_code;
	}
	public String getErr_code_des() {
		return err_code_des;
	}
	public void setErr_code_des(String err_code_des) {
		this.err_code_des = err_code_des;
	}
	public String getTrade_type() {
		return trade_type;
	}
	public void setTrade_type(String trade_type) {
		this.trade_type = trade_type;
	}
	public String getPrepay_id() {
		return prepay_id;
	}
	public void setPrepay_id(String prepay_id) {
		this.prepay_id = prepay_id;
	}
	public String getCode_url() {
		return code_url;
	}
	public void setCode_url(String code_url) {
		this.code_url = code_url;
	}
	public String getRs() {
		return rs;
	}
	public void setRs(String rs) {
		this.rs = rs;
	}
	
	
}




```


##### service

```

package com.sekorm.core.util.wx.service;


import com.sekorm.core.util.wx.util.HttpsRequest;
import com.sekorm.core.util.wx.util.HttpsRequestApp;

/**
 * User: rizenguo
 * Date: 2014/12/10
 * Time: 15:44
 * 服务的基类
 */
public class BaseService{

    //API的地址
    private String apiURL;

    //发请求的HTTPS请求器
    private HttpsRequest client;
    
  //发请求的HTTPS请求器
    private HttpsRequestApp client_app;

    public BaseService(String api) throws Exception {
        apiURL = api;
        client = new HttpsRequest();
    }

    protected String sendPost(Object xmlObj) throws Exception {
        return client.sendPost(apiURL, xmlObj);
    }
    
    public BaseService(String api,String app) throws Exception {
        apiURL = api;
        client_app = new HttpsRequestApp();
    }

    protected String sendPost(Object xmlObj,String app) throws Exception {
        return client_app.sendPost(apiURL, xmlObj);
    }
    
    
}


*****************************************************************


package com.sekorm.core.util.wx.service;

import com.sekorm.core.util.wx.request.DownloadBillReqData;
import com.sekorm.core.util.wx.util.Configure;

/**
 * User: rizenguo
 * Date: 2014/10/29
 * Time: 16:04
 */
public class DownloadBillService extends BaseService{

    public DownloadBillService() throws Exception {
        super(Configure.DOWNLOAD_BILL_API);
    }

    //ALL，返回当日所有订单信息，默认值
    public static final String BILL_TYPE_ALL = "ALL";

    //SUCCESS，返回当日成功支付的订单
    public static final String BILL_TYPE_SUCCESS = "SUCCESS";

    //REFUND，返回当日退款订单
    public static final String BILL_TYPE_REFUND = "REFUND";

    //REVOKED，已撤销的订单
    public static final String BILL_TYPE_REVOKE = "REVOKE";


    /**
     * 请求对账单下载服务
     * @param downloadBillReqData 这个数据对象里面包含了API要求提交的各种数据字段
     * @return API返回的XML数据
     * @throws Exception
     */
    public String request(DownloadBillReqData downloadBillReqData) throws Exception {

        //--------------------------------------------------------------------
        //发送HTTPS的Post请求到API地址
        //--------------------------------------------------------------------
        String responseString = sendPost(downloadBillReqData);

        return responseString;
    }

}



*****************************************************************************


package com.sekorm.core.util.wx.service;

import com.sekorm.core.util.wx.request.ScanPayQueryReqData;
import com.sekorm.core.util.wx.util.Configure;

/**
 * User: rizenguo
 * Date: 2014/10/29
 * Time: 16:04
 */
public class ScanPayQueryService extends BaseService{

    public ScanPayQueryService() throws Exception {
        super(Configure.PAY_QUERY_API);
    }

    /**
     * 请求支付查询服务
     * @param scanPayQueryReqData 这个数据对象里面包含了API要求提交的各种数据字段
     * @return API返回的XML数据
     * @throws Exception
     */
    public String request(ScanPayQueryReqData scanPayQueryReqData) throws Exception {

        //--------------------------------------------------------------------
        //发送HTTPS的Post请求到API地址
        //--------------------------------------------------------------------
        String responseString = sendPost(scanPayQueryReqData);

        return responseString;
    }


}



*******************************************************************************


package com.sekorm.core.util.wx.service;

import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;

import com.sekorm.core.util.wx.request.ScanPayReqData;
import com.sekorm.core.util.wx.util.Configure;
import com.sekorm.core.util.wx.util.RandomStringGenerator;
import com.sekorm.core.util.wx.util.Signature;

public class ScanPayService extends BaseService{

    public ScanPayService() throws Exception {
        super(Configure.PAY_API);
    }
    
    public ScanPayService(String app) throws Exception {
        super(Configure.PAY_API,app);
    }

    /**
     * 请求支付查询服务
     * @param scanPayReqData 这个数据对象里面包含了API要求提交的各种数据字段
     * @return API返回的XML数据
     * @throws Exception
     */
    public String request(ScanPayReqData scanPayReqData) throws Exception {

        //--------------------------------------------------------------------
        //发送HTTPS的Post请求到API地址
        //--------------------------------------------------------------------
        String responseString = sendPost(scanPayReqData);

        return responseString;
    }
    
    
    /**
     * 请求支付查询服务
     * @param scanPayReqData 这个数据对象里面包含了API要求提交的各种数据字段
     * @return API返回的XML数据
     * @throws Exception
     */
    public String request(ScanPayReqData scanPayReqData,String app) throws Exception {

        //--------------------------------------------------------------------
        //发送HTTPS的Post请求到API地址
        //--------------------------------------------------------------------
        String responseString = sendPost(scanPayReqData,app);

        return responseString;
    }
    
    

    public static void main(String[] args) {
    	ScanPayReqData data = new ScanPayReqData();
    	data.setBody("微信支付测试");
    	data.setOut_trade_no("w1704171234512");
    	data.setTotal_fee(1);
    	data.setSpbill_create_ip("172.16.33.48");
    	data.setNotify_url("https://172.16.33.48/notify");
    	data.setTrade_type("NATIVE");
    	data.setSign(Signature.getSign(data.toMap()));
    	System.out.println(data.toMap());
    	
    	try {
    		ScanPayService service = new ScanPayService();
//    		TestData data = new TestData();
//    		data.setSign(Signature.getSign(data.toMap()));
			String rs = service.request(data);
			System.out.println(rs);
		} catch (Exception e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
    	//https://api.mch.weixin.qq.com/sandboxnew/pay/getsignkey
//    	Util.getObjectFromXML(payServiceResponseString, ScanPayResData.class);
	}

}

class TestData {
	private String mch_id = "1288120701";
	private String nonce_str = RandomStringGenerator.getRandomStringByLength(32);
	private String sign = "";
	public String getMch_id() {
		return mch_id;
	}
	public void setMch_id(String mch_id) {
		this.mch_id = mch_id;
	}
	public String getNonce_str() {
		return nonce_str;
	}
	public void setNonce_str(String nonce_str) {
		this.nonce_str = nonce_str;
	}
	public String getSign() {
		return sign;
	}
	public void setSign(String sign) {
		this.sign = sign;
	}
	public Map<String,Object> toMap(){
		Map<String,Object> map = new HashMap<String, Object>();
		Field[] fields = this.getClass().getDeclaredFields();
		for (Field field : fields) {
			Object obj;
			try {
				obj = field.get(this);
				if(obj!=null){
					map.put(field.getName(), obj);
				}
			} catch (IllegalArgumentException e) {
				e.printStackTrace();
			} catch (IllegalAccessException e) {
				e.printStackTrace();
			}
		}
		return map;
	}
}








```



##### util


```

package com.sekorm.core.util.wx.util;



/**
 * User: rizenguo
 * Date: 2014/10/29
 * Time: 14:40
 * 这里放置各种配置数据
 */
public class Configure {

	//sdk的版本号
	public static   String sdkVersion ;

	//这个就是自己要保管好的私有Key了（切记只能放在自己的后台代码里，不能放在任何可能被看到源代码的客户端程序中）
	//每次自己Post数据给API的时候都要用这个key来对所有字段进行签名，生成的签名会放在Sign这个字段，API收到Post数据的时候也会用同样的签名算法对Post过来的数据进行签名和验证
	//收到API的返回的时候也要用这个key来对返回的数据算下签名，跟API的Sign数据进行比较，如果值不一致，有可能数据被第三方给篡改

	public static String key ;

	//微信分配的公众号ID（开通公众号之后可以获取到）wx45ee03a47e7019d0
	public static String appID  ;

	//微信支付分配的商户号ID（开通公众号的微信支付功能之后可以获取到）
	public static String mchID  ;

	//受理模式下给子商户分配的子商户号
	public static String subMchID  ;

	//HTTPS证书的本地路径
	public static String certLocalPath  ;

	//HTTPS证书密码，默认密码等于商户号MCHID
	public static String certPassword  ;

	//是否使用异步线程的方式来上报API测速，默认为异步模式
	public static boolean useThreadToDoReport  ;

	//机器IP
	public static String ip  ;
	
	public static String notify_url  ;
	
	public static String trade_type  ;
	
	

	//以下是几个API的路径：(https://api.mch.weixin.qq.com/sandboxnew/pay/getsignkey)
	//1）扫码支付API（统一下单）
	public static String PAY_API  ;

	//2）扫码支付查询API
	public static String PAY_QUERY_API  ;

	//3）关闭订单API
	public static String PAY_CLOSE_API ;

	//4）下载对账单API
	public static String DOWNLOAD_BILL_API  ;

	//5) 统计上报API
	public static String REPORT_API  ;

	
	//####################################
	public static String trade_type_app  ;
	public static String appID_app;
	public static String mchID_app  ;
	public static String key_app ;
	//HTTPS证书的本地路径
	public static String certLocalPath_app  ;
    //HTTPS证书密码，默认密码等于商户号MCHID
	public static String certPassword_app  ;
	
	public static String getSdkVersion() {
		return sdkVersion;
	}

	public static void setSdkVersion(String sdkVersion) {
		Configure.sdkVersion = sdkVersion;
	}

	public static String getKey() {
		return key;
	}

	public static void setKey(String key) {
		Configure.key = key;
	}

	public static String getAppID() {
		return appID;
	}

	public static void setAppID(String appID) {
		Configure.appID = appID;
	}

	public static String getMchID() {
		return mchID;
	}

	public static void setMchID(String mchID) {
		Configure.mchID = mchID;
	}

	public static String getSubMchID() {
		return subMchID;
	}

	public static void setSubMchID(String subMchID) {
		Configure.subMchID = subMchID;
	}

	public static String getCertLocalPath() {
		return certLocalPath;
	}

	public static void setCertLocalPath(String certLocalPath) {
		Configure.certLocalPath = certLocalPath;
	}

	public static String getCertPassword() {
		return certPassword;
	}

	public static void setCertPassword(String certPassword) {
		Configure.certPassword = certPassword;
	}

	public static boolean isUseThreadToDoReport() {
		return useThreadToDoReport;
	}

	public static void setUseThreadToDoReport(boolean useThreadToDoReport) {
		Configure.useThreadToDoReport = useThreadToDoReport;
	}

	public static String getIp() {
		return ip;
	}

	public static void setIp(String ip) {
		Configure.ip = ip;
	}

	public static String getPAY_API() {
		return PAY_API;
	}

	public static void setPAY_API(String pAY_API) {
		PAY_API = pAY_API;
	}

	public static String getPAY_QUERY_API() {
		return PAY_QUERY_API;
	}

	public static void setPAY_QUERY_API(String pAY_QUERY_API) {
		PAY_QUERY_API = pAY_QUERY_API;
	}

	public static String getPAY_CLOSE_API() {
		return PAY_CLOSE_API;
	}

	public static void setPAY_CLOSE_API(String pAY_CLOSE_API) {
		PAY_CLOSE_API = pAY_CLOSE_API;
	}

	public static String getDOWNLOAD_BILL_API() {
		return DOWNLOAD_BILL_API;
	}

	public static void setDOWNLOAD_BILL_API(String dOWNLOAD_BILL_API) {
		DOWNLOAD_BILL_API = dOWNLOAD_BILL_API;
	}

	public static String getREPORT_API() {
		return REPORT_API;
	}

	public static void setREPORT_API(String rEPORT_API) {
		REPORT_API = rEPORT_API;
	}

	public static String getNotify_url() {
		return notify_url;
	}

	public static void setNotify_url(String notify_url) {
		Configure.notify_url = notify_url;
	}

	public static String getTrade_type() {
		return trade_type;
	}

	public static void setTrade_type(String trade_type) {
		Configure.trade_type = trade_type;
	}

	public static String getTrade_type_app() {
		return trade_type_app;
	}

	public static void setTrade_type_app(String trade_type_app) {
		Configure.trade_type_app = trade_type_app;
	}

	public static String getAppID_app() {
		return appID_app;
	}

	public static void setAppID_app(String appID_app) {
		Configure.appID_app = appID_app;
	}

	public static String getMchID_app() {
		return mchID_app;
	}

	public static void setMchID_app(String mchID_app) {
		Configure.mchID_app = mchID_app;
	}

	public static String getKey_app() {
		return key_app;
	}

	public static void setKey_app(String key_app) {
		Configure.key_app = key_app;
	}

	public static String getCertLocalPath_app() {
		return certLocalPath_app;
	}

	public static void setCertLocalPath_app(String certLocalPath_app) {
		Configure.certLocalPath_app = certLocalPath_app;
	}

	public static String getCertPassword_app() {
		return certPassword_app;
	}

	public static void setCertPassword_app(String certPassword_app) {
		Configure.certPassword_app = certPassword_app;
	}

	
}



*****************************************************************************

package com.sekorm.core.util.wx.util;

import com.thoughtworks.xstream.XStream;
import com.thoughtworks.xstream.io.xml.DomDriver;
import com.thoughtworks.xstream.io.xml.XmlFriendlyNameCoder;
import org.apache.http.HttpEntity;
import org.apache.http.HttpResponse;
import org.apache.http.client.config.RequestConfig;
import org.apache.http.client.methods.HttpPost;
import org.apache.http.conn.ConnectTimeoutException;
import org.apache.http.conn.ConnectionPoolTimeoutException;
import org.apache.http.conn.ssl.SSLConnectionSocketFactory;
import org.apache.http.conn.ssl.SSLContexts;
import org.apache.http.entity.StringEntity;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.util.EntityUtils;
import org.slf4j.LoggerFactory;

import javax.net.ssl.SSLContext;
import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;
import java.net.SocketTimeoutException;
import java.security.KeyStore;
import java.security.NoSuchAlgorithmException;
import java.security.cert.CertificateException;

/**
 * User: rizenguo
 * Date: 2014/10/29
 * Time: 14:36
 */
public class HttpsRequest {

    public interface ResultListener {

        public void onConnectionPoolTimeoutError();
    }

    private static Log log = new Log(LoggerFactory.getLogger(HttpsRequest.class));

    //表示请求器是否已经做了初始化工作
    private boolean hasInit = false;

    //连接超时时间，默认10秒
    private int socketTimeout = 10000;

    //传输超时时间，默认30秒
    private int connectTimeout = 30000;

    //请求器的配置
    private RequestConfig requestConfig;

    //HTTP请求器
    private CloseableHttpClient httpClient;

    public HttpsRequest() throws Exception {
        init();
    }

    private void init() throws Exception {

        KeyStore keyStore = KeyStore.getInstance("PKCS12");
        FileInputStream instream = new FileInputStream(new File(Configure.getCertLocalPath()));//加载本地的证书进行https加密传输
        try {
            keyStore.load(instream, Configure.getCertPassword().toCharArray());//设置证书密码
        } catch (CertificateException e) {
            e.printStackTrace();
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        } finally {
            instream.close();
        }

        // Trust own CA and all self-signed certs
        SSLContext sslcontext = SSLContexts.custom()
                .loadKeyMaterial(keyStore, Configure.getCertPassword().toCharArray())
                .build();
        // Allow TLSv1 protocol only
        SSLConnectionSocketFactory sslsf = new SSLConnectionSocketFactory(
                sslcontext,
                new String[]{"TLSv1"},
                null,
                SSLConnectionSocketFactory.BROWSER_COMPATIBLE_HOSTNAME_VERIFIER);

        httpClient = HttpClients.custom()
                .setSSLSocketFactory(sslsf)
                .build();

        //根据默认超时限制初始化requestConfig
        requestConfig = RequestConfig.custom().setSocketTimeout(socketTimeout).setConnectTimeout(connectTimeout).build();

        hasInit = true;
    }

    /**
     * 通过Https往API post xml数据
     *
     * @param url    API地址
     * @param xmlObj 要提交的XML数据对象
     * @return API回包的实际数据
     * @throws IOException
     * @throws KeyStoreException
     * @throws UnrecoverableKeyException
     * @throws NoSuchAlgorithmException
     * @throws KeyManagementException
     */

    public String sendPost(String url, Object xmlObj) throws Exception {

        if (!hasInit) {
            init();
        }

        String result = null;

        HttpPost httpPost = new HttpPost(url);

        //解决XStream对出现双下划线的bug
        XStream xStreamForRequestPostData = new XStream(new DomDriver("UTF-8", new XmlFriendlyNameCoder("-_", "_")));

        //将要提交给API的数据对象转换成XML格式数据Post给API
        String postDataXML = xStreamForRequestPostData.toXML(xmlObj);

        Util.log("API，POST过去的数据是：");
        Util.log(postDataXML);

        //得指明使用UTF-8编码，否则到API服务器XML的中文不能被成功识别
        StringEntity postEntity = new StringEntity(postDataXML, "UTF-8");
        httpPost.addHeader("Content-Type", "text/xml");
        httpPost.setEntity(postEntity);

        //设置请求器的配置
        httpPost.setConfig(requestConfig);

        Util.log("executing request" + httpPost.getRequestLine());

        try {
            HttpResponse response = httpClient.execute(httpPost);

            HttpEntity entity = response.getEntity();

            result = EntityUtils.toString(entity, "UTF-8");

        } catch (ConnectionPoolTimeoutException e) {
            log.e("http get throw ConnectionPoolTimeoutException(wait time out)");

        } catch (ConnectTimeoutException e) {
            log.e("http get throw ConnectTimeoutException");

        } catch (SocketTimeoutException e) {
            log.e("http get throw SocketTimeoutException");

        } catch (Exception e) {
            log.e("http get throw Exception");

        } finally {
            httpPost.abort();
        }

        return result;
    }

    /**
     * 设置连接超时时间
     *
     * @param socketTimeout 连接时长，默认10秒
     */
    public void setSocketTimeout(int socketTimeout) {
    	this.socketTimeout = socketTimeout;
        resetRequestConfig();
    }

    /**
     * 设置传输超时时间
     *
     * @param connectTimeout 传输时长，默认30秒
     */
    public void setConnectTimeout(int connectTimeout) {
        this.connectTimeout = connectTimeout;
        resetRequestConfig();
    }

    private void resetRequestConfig(){
        requestConfig = RequestConfig.custom().setSocketTimeout(socketTimeout).setConnectTimeout(connectTimeout).build();
    }

    /**
     * 允许商户自己做更高级更复杂的请求器配置
     *
     * @param requestConfig 设置HttpsRequest的请求器配置
     */
    public void setRequestConfig(RequestConfig requestConfig) {
    	this.requestConfig = requestConfig;
    }
}



**********************************************************************


package com.sekorm.core.util.wx.util;

import com.thoughtworks.xstream.XStream;
import com.thoughtworks.xstream.io.xml.DomDriver;
import com.thoughtworks.xstream.io.xml.XmlFriendlyNameCoder;
import org.apache.http.HttpEntity;
import org.apache.http.HttpResponse;
import org.apache.http.client.config.RequestConfig;
import org.apache.http.client.methods.HttpPost;
import org.apache.http.conn.ConnectTimeoutException;
import org.apache.http.conn.ConnectionPoolTimeoutException;
import org.apache.http.conn.ssl.SSLConnectionSocketFactory;
import org.apache.http.conn.ssl.SSLContexts;
import org.apache.http.entity.StringEntity;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.util.EntityUtils;
import org.slf4j.LoggerFactory;

import javax.net.ssl.SSLContext;
import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;
import java.net.SocketTimeoutException;
import java.security.KeyStore;
import java.security.NoSuchAlgorithmException;
import java.security.cert.CertificateException;

/**
 * User: rizenguo
 * Date: 2014/10/29
 * Time: 14:36
 */
public class HttpsRequestApp {

    public interface ResultListener {

        public void onConnectionPoolTimeoutError();
    }

    private static Log log = new Log(LoggerFactory.getLogger(HttpsRequestApp.class));

    //表示请求器是否已经做了初始化工作
    private boolean hasInit = false;

    //连接超时时间，默认10秒
    private int socketTimeout = 10000;

    //传输超时时间，默认30秒
    private int connectTimeout = 30000;

    //请求器的配置
    private RequestConfig requestConfig;

    //HTTP请求器
    private CloseableHttpClient httpClient;

    public HttpsRequestApp() throws Exception {
        init();
    }

    private void init() throws Exception {

//        KeyStore keyStore = KeyStore.getInstance("PKCS12");
//        FileInputStream instream = new FileInputStream(new File(Configure.certLocalPath_app));//加载本地的证书进行https加密传输
//        try {
//            keyStore.load(instream, Configure.certPassword_app.toCharArray());//设置证书密码
//        } catch (CertificateException e) {
//            e.printStackTrace();
//        } catch (NoSuchAlgorithmException e) {
//            e.printStackTrace();
//        } finally {
//            instream.close();
//        }
//
//        // Trust own CA and all self-signed certs
//        SSLContext sslcontext = SSLContexts.custom()
//                .loadKeyMaterial(keyStore, Configure.certPassword_app.toCharArray())
//                .build();
//        // Allow TLSv1 protocol only
//        SSLConnectionSocketFactory sslsf = new SSLConnectionSocketFactory(
//                sslcontext,
//                new String[]{"TLSv1"},
//                null,
//                SSLConnectionSocketFactory.BROWSER_COMPATIBLE_HOSTNAME_VERIFIER);

//        httpClient = HttpClients.custom()
//                .setSSLSocketFactory(sslsf)
//                .build();
        
        httpClient = HttpClients.createDefault();
                

        //根据默认超时限制初始化requestConfig
        requestConfig = RequestConfig.custom().setSocketTimeout(socketTimeout).setConnectTimeout(connectTimeout).build();

        hasInit = true;
    }

    /**
     * 通过Https往API post xml数据
     *
     * @param url    API地址
     * @param xmlObj 要提交的XML数据对象
     * @return API回包的实际数据
     * @throws IOException
     * @throws KeyStoreException
     * @throws UnrecoverableKeyException
     * @throws NoSuchAlgorithmException
     * @throws KeyManagementException
     */

    public String sendPost(String url, Object xmlObj) throws Exception {

        if (!hasInit) {
            init();
        }

        String result = null;

        HttpPost httpPost = new HttpPost(url);

        //解决XStream对出现双下划线的bug
        XStream xStreamForRequestPostData = new XStream(new DomDriver("UTF-8", new XmlFriendlyNameCoder("-_", "_")));

        //将要提交给API的数据对象转换成XML格式数据Post给API
        String postDataXML = xStreamForRequestPostData.toXML(xmlObj);

//        Util.log("API，POST过去的数据是：");
//        Util.log(postDataXML);

        //得指明使用UTF-8编码，否则到API服务器XML的中文不能被成功识别
        StringEntity postEntity = new StringEntity(postDataXML, "UTF-8");
        httpPost.addHeader("Content-Type", "text/xml");
        httpPost.setEntity(postEntity);

        //设置请求器的配置
        httpPost.setConfig(requestConfig);

//        Util.log("executing request" + httpPost.getRequestLine());

        try {
            HttpResponse response = httpClient.execute(httpPost);

            HttpEntity entity = response.getEntity();

            result = EntityUtils.toString(entity, "UTF-8");

        } catch (ConnectionPoolTimeoutException e) {
            log.e("http get throw ConnectionPoolTimeoutException(wait time out)");

        } catch (ConnectTimeoutException e) {
            log.e("http get throw ConnectTimeoutException");

        } catch (SocketTimeoutException e) {
            log.e("http get throw SocketTimeoutException");

        } catch (Exception e) {
            log.e("http get throw Exception");

        } finally {
            httpPost.abort();
        }

        return result;
    }

    /**
     * 设置连接超时时间
     *
     * @param socketTimeout 连接时长，默认10秒
     */
    public void setSocketTimeout(int socketTimeout) {
    	this.socketTimeout = socketTimeout;
        resetRequestConfig();
    }

    /**
     * 设置传输超时时间
     *
     * @param connectTimeout 传输时长，默认30秒
     */
    public void setConnectTimeout(int connectTimeout) {
        this.connectTimeout = connectTimeout;
        resetRequestConfig();
    }

    private void resetRequestConfig(){
        requestConfig = RequestConfig.custom().setSocketTimeout(socketTimeout).setConnectTimeout(connectTimeout).build();
    }

    /**
     * 允许商户自己做更高级更复杂的请求器配置
     *
     * @param requestConfig 设置HttpsRequest的请求器配置
     */
    public void setRequestConfig(RequestConfig requestConfig) {
    	this.requestConfig = requestConfig;
    }
}




*************************************************************************

package com.sekorm.core.util.wx.util;

import org.slf4j.Logger;

/**
 * User: rizenguo
 * Date: 2014/11/12
 * Time: 14:32
 */
public class Log {

    public static final String LOG_TYPE_TRACE = "logTypeTrace";
    public static final String LOG_TYPE_DEBUG = "logTypeDebug";
    public static final String LOG_TYPE_INFO = "logTypeInfo";
    public static final String LOG_TYPE_WARN = "logTypeWarn";
    public static final String LOG_TYPE_ERROR = "logTypeError";

    //打印日志
    private Logger logger;

    public Log(Logger log){
        logger = log;
    }

    public void t(String s){
        logger.trace(s);
    }

    public void d(String s){
        logger.debug(s);
    }

    public void i(String s){
        logger.info(s);
    }

    public void w(String s){
        logger.warn(s);
    }

    public void e(String s){
        logger.error(s);
    }

    public void log(String type,String s){
        if(type.equals(Log.LOG_TYPE_TRACE)){
            t(s);
        }else if(type.equals(Log.LOG_TYPE_DEBUG)){
            d(s);
        }else if(type.equals(Log.LOG_TYPE_INFO)){
            i(s);
        }else if(type.equals(Log.LOG_TYPE_WARN)){
            w(s);
        }else if(type.equals(Log.LOG_TYPE_ERROR)){
            e(s);
        }
    }

}



*************************************************************************


package com.sekorm.core.util.wx.util;

import java.security.MessageDigest;

/**
 * User: rizenguo
 * Date: 2014/10/23
 * Time: 15:43
 */
public class MD5 {
    private final static String[] hexDigits = {"0", "1", "2", "3", "4", "5", "6", "7",
            "8", "9", "a", "b", "c", "d", "e", "f"};

    /**
     * 转换字节数组为16进制字串
     * @param b 字节数组
     * @return 16进制字串
     */
    public static String byteArrayToHexString(byte[] b) {
        StringBuilder resultSb = new StringBuilder();
        for (byte aB : b) {
            resultSb.append(byteToHexString(aB));
        }
        return resultSb.toString();
    }

    /**
     * 转换byte到16进制
     * @param b 要转换的byte
     * @return 16进制格式
     */
    private static String byteToHexString(byte b) {
        int n = b;
        if (n < 0) {
            n = 256 + n;
        }
        int d1 = n / 16;
        int d2 = n % 16;
        return hexDigits[d1] + hexDigits[d2];
    }

    /**
     * MD5编码
     * @param origin 原始字符串
     * @return 经过MD5加密之后的结果
     */
    public static String MD5Encode(String origin) {
        String resultString = null;
        try {
            resultString = origin;
            MessageDigest md = MessageDigest.getInstance("MD5");
            md.update(resultString.getBytes("UTF-8"));
            resultString = byteArrayToHexString(md.digest());
        } catch (Exception e) {
            e.printStackTrace();
        }
        return resultString;
    }

}




*********************************************************************


package com.sekorm.core.util.wx.util;

import java.util.Random;

/**
 * User: rizenguo
 * Date: 2014/10/29
 * Time: 14:18
 */
public class RandomStringGenerator {

    /**
     * 获取一定长度的随机字符串
     * @param length 指定字符串长度
     * @return 一定长度的字符串
     */
    public static String getRandomStringByLength(int length) {
        String base = "abcdefghijklmnopqrstuvwxyz0123456789";
        Random random = new Random();
        StringBuffer sb = new StringBuffer();
        for (int i = 0; i < length; i++) {
            int number = random.nextInt(base.length());
            sb.append(base.charAt(number));
        }
        return sb.toString();
    }

}




************************************************************************



package com.sekorm.core.util.wx.util;

import org.xml.sax.SAXException;

import javax.xml.parsers.ParserConfigurationException;
import java.io.IOException;
import java.lang.reflect.Field;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Map;

/**
 * User: rizenguo
 * Date: 2014/10/29
 * Time: 15:23
 */
public class Signature {
    /**
     * 签名算法
     * @param o 要参与签名的数据对象
     * @return 签名
     * @throws IllegalAccessException
     */
    public static String getSign(Object o) throws IllegalAccessException {
        ArrayList<String> list = new ArrayList<String>();
        Class cls = o.getClass();
        Field[] fields = cls.getDeclaredFields();
        for (Field f : fields) {
            f.setAccessible(true);
            if (f.get(o) != null && f.get(o) != "") {
                list.add(f.getName() + "=" + f.get(o) + "&");
            }
        }
        int size = list.size();
        String [] arrayToSort = list.toArray(new String[size]);
        Arrays.sort(arrayToSort, String.CASE_INSENSITIVE_ORDER);
        StringBuilder sb = new StringBuilder();
        for(int i = 0; i < size; i ++) {
            sb.append(arrayToSort[i]);
        }
        String result = sb.toString();
        result += "key=" + Configure.getKey();
        Util.log("Sign Before MD5:" + result);
        result = MD5.MD5Encode(result).toUpperCase();
        Util.log("Sign Result:" + result);
        return result;
    }

    public static String getSign(Map<String,Object> map){
        ArrayList<String> list = new ArrayList<String>();
        for(Map.Entry<String,Object> entry:map.entrySet()){
            if(entry.getValue()!=""){
                list.add(entry.getKey() + "=" + entry.getValue() + "&");
            }
        }
        int size = list.size();
        String [] arrayToSort = list.toArray(new String[size]);
        Arrays.sort(arrayToSort, String.CASE_INSENSITIVE_ORDER);
        StringBuilder sb = new StringBuilder();
        for(int i = 0; i < size; i ++) {
            sb.append(arrayToSort[i]);
        }
        String result = sb.toString();
        result += "key=" + Configure.getKey();
        //Util.log("Sign Before MD5:" + result);
        result = MD5.MD5Encode(result).toUpperCase();
        //Util.log("Sign Result:" + result);
        return result;
    }
    
    
    
    public static String getSignApp(Object o) throws IllegalAccessException {
        ArrayList<String> list = new ArrayList<String>();
        Class cls = o.getClass();
        Field[] fields = cls.getDeclaredFields();
        for (Field f : fields) {
            f.setAccessible(true);
            if (f.get(o) != null && f.get(o) != "") {
                list.add(f.getName() + "=" + f.get(o) + "&");
            }
        }
        int size = list.size();
        String [] arrayToSort = list.toArray(new String[size]);
        Arrays.sort(arrayToSort, String.CASE_INSENSITIVE_ORDER);
        StringBuilder sb = new StringBuilder();
        for(int i = 0; i < size; i ++) {
            sb.append(arrayToSort[i]);
        }
        String result = sb.toString();
        result += "key=" + Configure.key_app;
        Util.log("Sign Before MD5:" + result);
        result = MD5.MD5Encode(result).toUpperCase();
        Util.log("Sign Result:" + result);
        return result;
    }
    
    
    public static String getSignApp(Map<String,Object> map){
        ArrayList<String> list = new ArrayList<String>();
        for(Map.Entry<String,Object> entry:map.entrySet()){
            if(entry.getValue()!=""){
                list.add(entry.getKey() + "=" + entry.getValue() + "&");
            }
        }
        int size = list.size();
        String [] arrayToSort = list.toArray(new String[size]);
        Arrays.sort(arrayToSort, String.CASE_INSENSITIVE_ORDER);
        StringBuilder sb = new StringBuilder();
        for(int i = 0; i < size; i ++) {
            sb.append(arrayToSort[i]);
        }
        String result = sb.toString();
        result += "key=" + Configure.key_app;
        //Util.log("Sign Before MD5:" + result);
        result = MD5.MD5Encode(result).toUpperCase();
        //Util.log("Sign Result:" + result);
        return result;
    }
    
    

    /**
     * 从API返回的XML数据里面重新计算一次签名
     * @param responseString API返回的XML数据
     * @return 新鲜出炉的签名
     * @throws ParserConfigurationException
     * @throws IOException
     * @throws SAXException
     */
    public static String getSignFromResponseString(String responseString) throws IOException, SAXException, ParserConfigurationException {
        Map<String,Object> map = XMLParser.getMapFromXML(responseString);
        //清掉返回数据对象里面的Sign数据（不能把这个数据也加进去进行签名），然后用签名算法进行签名
        map.put("sign","");
        //将API返回的数据根据用签名算法进行计算新的签名，用来跟API返回的签名进行比较
        return Signature.getSign(map);
    }

    /**
     * 检验API返回的数据里面的签名是否合法，避免数据在传输的过程中被第三方篡改
     * @param responseString API返回的XML数据字符串
     * @return API签名是否合法
     * @throws ParserConfigurationException
     * @throws IOException
     * @throws SAXException
     */
    public static boolean checkIsSignValidFromResponseString(String responseString) throws ParserConfigurationException, IOException, SAXException {

        Map<String,Object> map = XMLParser.getMapFromXML(responseString);
        Util.log(map.toString());

        String signFromAPIResponse = map.get("sign").toString();
        if(signFromAPIResponse=="" || signFromAPIResponse == null){
            Util.log("API返回的数据签名数据不存在，有可能被第三方篡改!!!");
            return false;
        }
        Util.log("服务器回包里面的签名是:" + signFromAPIResponse);
        //清掉返回数据对象里面的Sign数据（不能把这个数据也加进去进行签名），然后用签名算法进行签名
        map.put("sign","");
        //将API返回的数据根据用签名算法进行计算新的签名，用来跟API返回的签名进行比较
        String signForAPIResponse = Signature.getSign(map);

        if(!signForAPIResponse.equals(signFromAPIResponse)){
            //签名验不过，表示这个API返回的数据有可能已经被篡改了
            Util.log("API返回的数据签名验证不通过，有可能被第三方篡改!!!");
            return false;
        }
        Util.log("恭喜，API返回的数据签名验证通过!!!");
        return true;
    }

}




****************************************************************************************


package com.sekorm.core.util.wx.util;

import com.thoughtworks.xstream.XStream;
import org.slf4j.LoggerFactory;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.UnsupportedEncodingException;
import java.lang.reflect.Field;
import java.util.Map;

/**
 * User: rizenguo
 * Date: 2014/10/23
 * Time: 14:59
 */
public class Util {

    //打log用
    private static Log logger = new Log(LoggerFactory.getLogger(Util.class));

    /**
     * 通过反射的方式遍历对象的属性和属性值，方便调试
     *
     * @param o 要遍历的对象
     * @throws Exception
     */
    public static void reflect(Object o) throws Exception {
        Class cls = o.getClass();
        Field[] fields = cls.getDeclaredFields();
        for (int i = 0; i < fields.length; i++) {
            Field f = fields[i];
            f.setAccessible(true);
            Util.log(f.getName() + " -> " + f.get(o));
        }
    }

    public static byte[] readInput(InputStream in) throws IOException {
        ByteArrayOutputStream out = new ByteArrayOutputStream();
        int len = 0;
        byte[] buffer = new byte[1024];
        while ((len = in.read(buffer)) > 0) {
            out.write(buffer, 0, len);
        }
        out.close();
        in.close();
        return out.toByteArray();
    }

    public static String inputStreamToString(InputStream is) throws IOException {
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        int i;
        while ((i = is.read()) != -1) {
            baos.write(i);
        }
        return baos.toString();
    }


    public static InputStream getStringStream(String sInputString) throws UnsupportedEncodingException {
        ByteArrayInputStream tInputStringStream = null;
        if (sInputString != null && !sInputString.trim().equals("")) {
            tInputStringStream = new ByteArrayInputStream(sInputString.getBytes("UTF-8"));
        }
        return tInputStringStream;
    }

    public static Object getObjectFromXML(String xml, Class tClass) {
        //将从API返回的XML数据映射到Java对象
        XStream xStreamForResponseData = new XStream();
        xStreamForResponseData.alias("xml", tClass);
        xStreamForResponseData.ignoreUnknownElements();//暂时忽略掉一些新增的字段
        return xStreamForResponseData.fromXML(xml);
    }

    public static String getStringFromMap(Map<String, Object> map, String key, String defaultValue) {
        if (key == "" || key == null) {
            return defaultValue;
        }
        String result = (String) map.get(key);
        if (result == null) {
            return defaultValue;
        } else {
            return result;
        }
    }

    public static int getIntFromMap(Map<String, Object> map, String key) {
        if (key == "" || key == null) {
            return 0;
        }
        if (map.get(key) == null) {
            return 0;
        }
        return Integer.parseInt((String) map.get(key));
    }

    /**
     * 打log接口
     * @param log 要打印的log字符串
     * @return 返回log
     */
    public static String log(Object log){
        logger.i(log.toString());
        //System.out.println(log);
        return log.toString();
    }

    /**
     * 读取本地的xml数据，一般用来自测用
     * @param localPath 本地xml文件路径
     * @return 读到的xml字符串
     */
    public static String getLocalXMLString(String localPath) throws IOException {
        return Util.inputStreamToString(Util.class.getResourceAsStream(localPath));
    }

}




************************************************************************************


package com.sekorm.core.util.wx.util;

import org.w3c.dom.Document;
import org.w3c.dom.Element;
import org.w3c.dom.Node;
import org.w3c.dom.NodeList;
import org.xml.sax.SAXException;

import javax.xml.parsers.DocumentBuilder;
import javax.xml.parsers.DocumentBuilderFactory;
import javax.xml.parsers.ParserConfigurationException;
import java.io.IOException;
import java.io.InputStream;
import java.util.HashMap;
import java.util.Map;

/**
 * User: rizenguo
 * Date: 2014/11/1
 * Time: 14:06
 */
public class XMLParser {

    public static Map<String,Object> getMapFromXML(String xmlString) throws ParserConfigurationException, IOException, SAXException {

        //这里用Dom的方式解析回包的最主要目的是防止API新增回包字段
        DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
        DocumentBuilder builder = factory.newDocumentBuilder();
        InputStream is =  Util.getStringStream(xmlString);
        Document document = builder.parse(is);

        //获取到document里面的全部结点
        NodeList allNodes = document.getFirstChild().getChildNodes();
        Node node;
        Map<String, Object> map = new HashMap<String, Object>();
        int i=0;
        while (i < allNodes.getLength()) {
            node = allNodes.item(i);
            if(node instanceof Element){
                map.put(node.getNodeName(),node.getTextContent());
            }
            i++;
        }
        return map;

    }


}



*********************************************************************************



//package com.sekorm.core.util.wx.util;
//
//import com.google.zxing.BarcodeFormat;
//import com.google.zxing.EncodeHintType;
//import com.google.zxing.MultiFormatWriter;
//import com.google.zxing.common.BitMatrix;
//import com.google.zxing.qrcode.decoder.ErrorCorrectionLevel;
//
//import org.apache.commons.logging.Log;
//import org.apache.commons.logging.LogFactory;
//
//import javax.imageio.ImageIO;
//
//import java.awt.image.BufferedImage;
//import java.io.IOException;
//import java.io.OutputStream;
//import java.util.Hashtable;
//import java.util.Map;
//
///**
// * Created by liuyangkly on 15/6/27.
// * 使用了google zxing作为二维码生成工具
// */
//public class ZxingUtils {
//	private static Log log = LogFactory.getLog(ZxingUtils.class);
//
//    private static final int BLACK = 0xFF000000;
//    private static final int WHITE = 0xFFFFFFFF;
//
//    private static BufferedImage toBufferedImage(BitMatrix matrix) {
//        int width = matrix.getWidth();
//        int height = matrix.getHeight();
//        BufferedImage image = new BufferedImage(width, height, BufferedImage.TYPE_INT_RGB);
//        for (int x = 0; x < width; x++) {
//            for (int y = 0; y < height; y++) {
//                image.setRGB(x, y, matrix.get(x, y) ? BLACK : WHITE);
//            }
//        }
//        return image;
//    }
//
//    private static void writeToFile(BitMatrix matrix, String format, OutputStream file) throws IOException {
//        BufferedImage image = toBufferedImage(matrix);
//        if (!ImageIO.write(image, format, file)) {
//            throw new IOException("Could not write an image of format " + format + " to " + file);
//        }
//    }
//
//    /** 将内容contents生成长宽均为width的图片，图片路径由imgPath指定
//     */
//    public static void setQRCodeImge(String contents, int width, OutputStream file) {
//        setQRCodeImge(contents, width, width, file);
//    }
//
//    /** 将内容contents生成长为width，宽为width的图片，图片路径由imgPath指定
//     */
//	public static void setQRCodeImge(String contents, int width, int height, OutputStream file) {
//		try {
//            Map<EncodeHintType, Object> hints = new Hashtable<EncodeHintType, Object>();
//            hints.put(EncodeHintType.ERROR_CORRECTION, ErrorCorrectionLevel.M);
//            hints.put(EncodeHintType.CHARACTER_SET, "UTF8");
//
//			BitMatrix bitMatrix = new MultiFormatWriter().encode(contents, BarcodeFormat.QR_CODE, width, height, hints);
//
//			writeToFile(bitMatrix, "png", file);
//		} catch (Exception e) {
//			log.error("create QR code error!", e);
//		}
//	}
//}















```