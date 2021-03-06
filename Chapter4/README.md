---
title: Big Rob机器人小车项目五:差分GPS(differential GPS-DGPS，DGPS)
date: 2019-12-20 16:18:07
tags: 机器人小车
categories: 
- 内在提升
- 技能爱好
- 第二技能
---
机器人小车项目尚未结束。我们已经安装了电子组件。该机器人的大脑是Raspberry Pi 2 ModelB。RaspberryPi连接到RasPiGNSS模块以接收原始GPS数据流。为了通过WIFI控制Big Rob，Raspberry Pi通过RJ45电缆连接到机身内置的Linksys路由器。但是我将来会更改此设置，并将安装带有增强器和外部天线的USB wifi卡。机器人可以自主导航，并可以通过带有实时视频流的Web界面进行手动控制，以向操作员显示机器人看到的内容。

下一张照片显示了Big Rob准备在野外执行任务的情况。在机器人的顶部，安装了Raspberry Pi Sense-HAT和GPS天线。机器人需要连接到RasPiGNSS模块的GPS天线，以进行精确差分的GPS导航。在此设置中，GPS所需的基站没有显示在图片上。但是Big Rob的Raspberry Pi是通过WIFI连接到基站的，RTK库已经使用机器人的GPS精确坐标计算出FIX解决方案。

![树莓派机器人车](http://yuntu88.oss-cn-beijing.aliyuncs.com/fromlocal/1242937438@qq.com/20191220/sGQErxsnaH.jpg)

Big Rob本身使用Linksys WRT54GL WIFI路由器，该路由器用作访问点来控制机器人并与基站连接。机器人内部的路由器使准备执行任务的机器人非常方便。WIFI连接非常可靠，这对操作员来说很重要。
<!-- more -->
#### 具有GNSS导航功能的RASPBERRY PI机器人

自2016年11月以来，我一直在为的机器人汽车开发GPS解决方案。RTK解决方案由基站和内置在机器人中的移动单元组成。如果您想了解有关GPS解决方案设置的更多信息，我写了一篇有关使用RTK库的差分GPS解决方案的安装和使用的文章系列。

链接：[Raspberry Pi和RTKLIB的GNSS-Positionierung MIT –Einführung](https://custom-build-robots.com/elektronik/praezise-gnss-positionierung-mit-dem-raspberry-pi-und-rtklib-einfuehrung/7921?lang=de)

下图显示了机器人内部内置的技术组件的一小部分。在左侧，您会看到RasPiGNSS GPS接收器，在RasPiGNSS模块的顶部，您会看到GPS天线。

![Raspberry Pi-Big Rob RasPiGNSS SenseHAT](http://yuntu88.oss-cn-beijing.aliyuncs.com/fromlocal/1242937438@qq.com/20191220/QfzmWCY3Em.jpg)


在尾部，我安装了Raspberry Pi Sense-HAT。需要Sense-HAT借助内置磁力计将机器人的前部转向北方。使用Sense-HAT内的陀螺仪，可以控制机器人必须向左或向右旋转多远才能面向北或要驶向的GPS航路点。

#### PYTHON模块化GPS NMEA数据集

过去，我使用USB GPS鼠标进行导航。GPSD守护程序和Python GPS模块支持大多数USB GPS模块。这种GPS鼠标的典型代表是NMEA流，可以从USB接收器所连接的USB接口读取该流。GPSD守护程序会读取此NEMA数据流，以在我的Python程序中进行进一步处理。

对于GPS RTK解决方案，我更改了Python程序以从网络上的TCP / IP套接字而不是USB端口读取NMEA流。这使Python程序能够读取RTKRCV程序生成的非常精确的NMEA流。我的机器人可以通过RTK解决方案以计算出的位置非常精确地导航。

更改Python程序非常容易。可以从TCP / IP套接字直接通过网络读取新的NMEA流，并使用扩展名PYNMEA轻松地在Python程序中处理NMEA数据。使用PYNMEA，我可以解释RTKRCV生成的NMEA流，并使用GPS信息来控制机器人。

![Python读取RTK流](http://yuntu88.oss-cn-beijing.aliyuncs.com/fromlocal/1242937438@qq.com/20191220/meiS7aFKkA.jpg)

要在Raspberry Pi上安装PYNMEA，请在Raspbian操作系统的终端窗口中执行以下命令：

`sudo pip install pynmea`

可以从以下URL下载读取NMEA流的Python程序。该程序是进一步程序的基础，并总体上说明了如何使用Python处理GPS数据以控制机器人。

下载： [阅读RTKRCV NMEA PYTHON流](https://custom-build-robots.com/wp-content/uploads/2017/02/READ_RTKRCV_NMEA_PYTHON_STREAM.zip)

```
 #!/usr/bin/env python
# coding: latin-1
# Autor:   Ingmar Stapel
# Datum:   20170225
# Version:   1.0
# Homepage:   http://custom-build-robots.com
# This program was developed to read the NMEA stream
# generated by the RTK library for precise navigation.

# Parts of the program regarding the nmea processing are from
# the following website:
# https://amalgjose.com/tag/python-program-to-read-gps-values/
# To run this program you have to install pynmea on your machine.
# pynmea can be installed with pip by typing:
# sudo pip install pynmea

import os
from gps import *
import string
from pynmea import nmea

import socket
import sys

# Create a TCP/IP socket
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# Define the server address and port from which the NMEA
# should be read from.
server_address = ('localhost', 2948)

try:
    # Connect the socket to the port where the RTK server is listening
    sock.connect(server_address)
    print '-------------------------------------------'    
    print >>sys.stderr, 'connecting to <%s> port <%s>' % server_address            
    print '-------------------------------------------'
except:
    print "Unexpected error:", sys.exc_info()[0]
    raise

# Read each line from the NMEA stream.
def readlines(sock, recv_buffer=4096, delim='\n'):
    buffer = ''
    data = True
    while data:
        data = sock.recv(recv_buffer)
        buffer += data

        while buffer.find(delim) != -1:
            line, buffer = buffer.split('\n', 1)
            yield line
    return

# Create an instance of an GPGGA object.     
gpgga = nmea.GPGGA()

# Process the NMEA data and display the result.    
try:
    for line in readlines(sock):
        #print line
        if line[0:6] == '$GPGGA':
            os.system('clear')
            
            #method for parsing the NMEA sentence
            gpgga.parse(line)
            lats = gpgga.latitude
            
            # Print some information about the position of the
            # mobile station / robot.
            print '-------------------------------------------'    
            print >>sys.stderr, 'connecting to <%s> port <%s>' % server_address        
            print '-------------------------------------------'
            print '\n-------------------------------------------'
            print '          READ RTK NMEA STREAM             '
            print '-------------------------------------------'            
            print "Latitude values : " + str(lats)

            lat_dir = gpgga.lat_direction
            print "Latitude direction : " + str(lat_dir)

            longitude = gpgga.longitude
            print "Longitude values : " + str(longitude)

            long_dir = gpgga.lon_direction
            print "Longitude direction : " + str(long_dir)

            time_stamp = gpgga.timestamp
            print "GPS time stamp : " + str(time_stamp)

            alt = gpgga.antenna_altitude
            print "Antenna altitude : " + str(alt)
            print '-------------------------------------------'
            
            lats = gpgga.latitude
            longs = gpgga.longitude
            
            #convert degrees,decimal minutes to decimal degrees
            # The source for the decimal calcualtion was:
            # http://dlnmh9ip6v2uc.cloudfront.net/tutorialimages/Python_and_GPS/gpsmap.py
            lat1 = (float(lats[2]+lats[3]+lats[4]+lats[5]+lats[6]+lats[7]+lats[8]))/60
            lat = (float(lats[0]+lats[1])+lat1)
            long1 = (float(longs[3]+longs[4]+longs[5]+longs[6]+longs[7]+longs[8]+longs[9]))/60
            long = (float(longs[0]+longs[1]+longs[2])+long1)            
            
            print '\n------------ decimal degrees --------------'    
            print 'lat: ',lat
            print 'long: ',long
            print '-------------------------------------------'
except:
    print "Unexpected error:", sys.exc_info()[0]
    raise

# Program end

```

#### 总结：

现在，我对Python程序中的更改感到满意。我已经使用精确GPS装置在野外测试了Big Rob。技术上的非常努力最终为的机器人获得一个非常精确的定位系统，那是值得的。花了一段时间才能自动启动并运行所有内容。但是就目前而言，RTK解决方案工作得很好，我只需要给基站和移动单元加电即可获得非常精确的NMEA流进行导航。

