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
