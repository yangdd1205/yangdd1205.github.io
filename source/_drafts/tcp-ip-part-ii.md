---
title: TCP/IP 基础
categories: 计算机网络
tags:
  - 计算机网络
  - TCP/IP
---
> 本篇内容来源于《图解TCP/IP》第二章

上一篇介绍了网络基础知识，这一篇说说 TCP/IP 基础知识。

<!-- more -->

## TCP/IP 与 OSI 参考模型

<table>
    <tr>
        <th>OSI 参考模型</th>
        <th colspan="2">TCP/IP 分层模型</th>
    </tr>
    <tr>
        <td>应用层</td>
        <td rowspan="3">应用层<br>DNS、URI、HTML、HTTP<br>TLS/SSL、SMTP、POP、IMAP<br>MIME、TELNET、SSH、FTP<br>SNMP、MIB、SIP、RTP、LDAP</td>
        <td rowspan="3">应用程序</td>
    </tr>
    <tr>
        <td>表示层</td>
    </tr>
    <tr>
        <td>会话层</td>
    </tr>
    <tr>
        <td>传输层</td>
        <td>传输层<br>TCP、UDP、UDP-Lite、SCTP、DCCP</td>
        <td rowspan="2">操作系统</td>
    </tr>
    <tr>
        <td>网络层</td>
        <td>互联网层<br>ARP、IP、ICMP</td>
    </tr>
    <tr>
        <td>数据链路层</td>
        <td>网卡层</td>
        <td rowspan="2">设备驱动程序与网络接口</td>
    </tr>
    <tr>
        <td>物理层</td>
        <td>（硬件）</td>
    </tr>
</table>

