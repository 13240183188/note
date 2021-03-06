https://mp.weixin.qq.com/s/Iy1uxCeImyUUTx2fwI6_qA

作者：宇的季节
cnblogs.com/chenkeyu/p/8001624.html
前言
最近在项目中需要和ftp服务器进行交互，在网上找了一下关于ftp上传下载的工具类，大致有两种。
第一种是单例模式的类。
第二种是另外定义一个Service，直接通过Service来实现ftp的上传下载删除。
这两种感觉都有利弊。
第一种实现了代码复用，但是配置信息全需要写在类中，维护比较复杂。
第二种如果是spring框架，可以通过propertis文件，动态的注入配置信息，但是又不能代码复用。
所以我打算自己实现一个工具类，来把上面的两种优点进行整合。顺便把一些上传过程中一些常见的问题也给解决了。
因为我使用的是spring框架，如果把工具类声明为bean给spring管理，他默认就是单例的，所以不需要我再实现单例。并且因为是bean，所以我可以把properties文件的属性注入bean的属性中，实现解耦，下面是具体代码：
package com.cky.util;

import java.io.File;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;

import org.apache.commons.net.ftp.FTP;
import org.apache.commons.net.ftp.FTPClient;
import org.apache.commons.net.ftp.FTPFile;
import org.apache.commons.net.ftp.FTPReply;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

//使用spring自动生成单例对象，
//@Component
public class FtpUtil {
//通过properties文件自动注入
@Value("${ftp.host}")
private String host;    //ftp服务器ip
@Value("${ftp.port}")
private int port;        //ftp服务器端口
@Value("${ftp.username}")
private String username;//用户名
@Value("${ftp.password}")
private String password;//密码
@Value("${ftp.basePath}")
private String basePath;//存放文件的基本路径
//测试的时候把这个构造函数打开，设置你的初始值，然后在代码后面的main方法运行测试
public FtpUtil() {
//System.out.println(this.toString());
host="192.168.100.77";
port=21;
username="ftpuser";
password="ftp54321";
basePath="/home/ftpuser/www/images";
}
/**
*
* @param path        上传文件存放在服务器的路径
* @param filename    上传文件名
* @param input        输入流
* @return
*/
public boolean fileUpload(String path,String filename,InputStream input) {
FTPClient ftp=new FTPClient();
try {
ftp.connect(host, port);
ftp.login(username, password);
//设置文件编码格式
ftp.setControlEncoding("UTF-8");
//ftp通信有两种模式
//PORT(主动模式)客户端开通一个新端口(>1024)并通过这个端口发送命令或传输数据,期间服务端只使用他开通的一个端口，例如21
//PASV(被动模式)客户端向服务端发送一个PASV命令，服务端开启一个新端口(>1024),并使用这个端口与客户端的21端口传输数据
//由于客户端不可控，防火墙等原因，所以需要由服务端开启端口，需要设置被动模式
ftp.enterLocalPassiveMode();
//设置传输方式为流方式
ftp.setFileTransferMode(FTP.STREAM_TRANSFER_MODE);
//获取状态码，判断是否连接成功
if(!FTPReply.isPositiveCompletion(ftp.getReplyCode())) {
throw new RuntimeException("FTP服务器拒绝连接");
}
//转到上传文件的根目录
if(!ftp.changeWorkingDirectory(basePath)) {
throw new RuntimeException("根目录不存在，需要创建");
}
//判断是否存在目录
if(!ftp.changeWorkingDirectory(path)) {
String[] dirs=path.split("/");
//创建目录
for (String dir : dirs) {
if(null==dir||"".equals(dir)) continue;
//判断是否存在目录
if(!ftp.changeWorkingDirectory(dir)) {
//不存在则创建
if(!ftp.makeDirectory(dir)) {
throw new RuntimeException("子目录创建失败");
}
//进入新创建的目录
ftp.changeWorkingDirectory(dir);
}
}
//设置上传文件的类型为二进制类型
ftp.setFileType(FTP.BINARY_FILE_TYPE);
//上传文件
if(!ftp.storeFile(filename, input)) {
return false;
}
input.close();
ftp.logout();
return true;
}


        } catch (Exception e) {
            throw new RuntimeException(e);
        }finally {
            if(ftp.isConnected()) {
                try {
                    ftp.disconnect();
                } catch (IOException e) {
                    throw new RuntimeException(e);
                }
            }
        }
        return false;
    }
    /**
     * 
     * @param filename    文件名，注意！此处文件名为加路径文件名，如：/2015/06/04/aa.jpg
     * @param localPath    存放到本地第地址
     * @return        
     */
    public boolean downloadFile(String filename,String localPath) {
        FTPClient ftp=new FTPClient();
        try {
            ftp.connect(host, port);
            ftp.login(username, password);
            //设置文件编码格式
            ftp.setControlEncoding("UTF-8");
            //ftp通信有两种模式
                //PORT(主动模式)客户端开通一个新端口(>1024)并通过这个端口发送命令或传输数据,期间服务端只使用他开通的一个端口，例如21
                //PASV(被动模式)客户端向服务端发送一个PASV命令，服务端开启一个新端口(>1024),并使用这个端口与客户端的21端口传输数据
                //由于客户端不可控，防火墙等原因，所以需要由服务端开启端口，需要设置被动模式
            ftp.enterLocalPassiveMode();
            //设置传输方式为流方式
            ftp.setFileTransferMode(FTP.STREAM_TRANSFER_MODE);
            //获取状态码，判断是否连接成功
            if(!FTPReply.isPositiveCompletion(ftp.getReplyCode())) {
                throw new RuntimeException("FTP服务器拒绝连接");
            }
            
            int index=filename.lastIndexOf("/");
            //获取文件的路径
            String path=filename.substring(0, index);
            //获取文件名
            String name=filename.substring(index+1);
            //判断是否存在目录
            if(!ftp.changeWorkingDirectory(basePath+path)) {
                throw new RuntimeException("文件路径不存在："+basePath+path);
            }
            //获取该目录所有文件
            FTPFile[] files=ftp.listFiles();
            for (FTPFile file : files) {
                //判断是否有目标文件
                //System.out.println("文件名"+file.getName()+"---"+name);
                if(file.getName().equals(name)) {
                    //System.out.println("找到文件");
                    //如果找到，将目标文件复制到本地
                    File localFile =new File(localPath+"/"+file.getName());
                    OutputStream out=new FileOutputStream(localFile);
                    ftp.retrieveFile(file.getName(), out);
                    out.close();
                }
            }
            ftp.logout();
            return true;
        } catch (Exception e) {
            throw new RuntimeException(e);
        }finally {
            if(ftp.isConnected()) {
                try {
                    ftp.disconnect();
                } catch (IOException e) {
                    throw new RuntimeException(e);
                }
            }
        }
    }
    
    public boolean deleteFile(String filename) {
        FTPClient ftp=new FTPClient();
        try {
            ftp.connect(host, port);
            ftp.login(username, password);
            //设置编码格式
            ftp.setControlEncoding("UTF-8");
            ftp.enterLocalPassiveMode();
            //获取状态码，判断是否连接成功
            if(!FTPReply.isPositiveCompletion(ftp.getReplyCode())) {
                throw new RuntimeException("FTP服务器拒绝连接");
            }
            int index=filename.lastIndexOf("/");
            //获取文件的路径
            String path=filename.substring(0, index);
            //获取文件名
            String name=filename.substring(index+1);
            //判断是否存在目录,不存在则说明文件存在
            if(!ftp.changeWorkingDirectory(basePath+path)) {
                return true;
            }
            if(ftp.deleteFile(name)) {
                clearDirectory(ftp, basePath, path);
                ftp.logout();
                return true;
            }
            ftp.logout();
            return false;
        } catch (Exception e) {
            throw new RuntimeException(e);
        }finally {
            if(ftp.isConnected()) {
                try {
                    ftp.disconnect();
                } catch (IOException e) {
                    throw new RuntimeException(e);
                }
            }
        }
    }
    
    /**
     * 
     * @param ftp    
     * @param basePath    
     * @param path        以path为根，递归清除上面所有空的文件夹，直到出现不为空的文件夹停止，最多清除到basePath结束
     * @throws IOException
     */
    private void clearDirectory(FTPClient ftp,String basePath,String path) throws IOException {
        //如果路径长度小于2，说明到顶了
        if(path.length()<2) {
            return ;
        }
        //如果当前目录文件数目小于1则删除此目录
        if(ftp.listNames(basePath+path).length<1) {
            //删除目录
            System.out.println("删除目录："+basePath+path);
            ftp.removeDirectory(basePath+path);
            int index=path.lastIndexOf("/");
            //路径向上一层
            path=path.substring(0, index);
            //继续判断
            clearDirectory(ftp, basePath, path);
        }
    }
    
    //两个功能其中一个使用的话另一个需要注释
    public static void main(String []args) {
        //上传测试--------------------------------------
        /*FileInputStream in;
        try {
            in=new FileInputStream(new File("C:\\Users\\Administrator\\Desktop\\json.png"));
            FtpUtil ftputil=new FtpUtil();
            boolean flag=ftputil.fileUpload("/2015/06/04", "va.jpg", in);
            System.out.println(flag);
        }catch (Exception e) {
            e.printStackTrace();
        }finally {
        }*/
        //下载测试--------------------------------------
        /*String filename="/2015/06/04/aa.jpg";
        String localPath="F:\\";
        FtpUtil ftputil=new FtpUtil();
        ftputil.downloadFile(filename, localPath);*/
        //删除测试--------------------------------------
        FtpUtil ftputil=new FtpUtil();
        boolean flag=ftputil.deleteFile("/2015/06/04/va.jpg");
        System.out.println(flag);
    }
    
    public String getHost() {
        return host;
    }
    public void setHost(String host) {
        this.host = host;
    }
    public int getPort() {
        return port;
    }
    public void setPort(int port) {
        this.port = port;
    }
    public String getUsername() {
        return username;
    }
    public void setUsername(String username) {
        this.username = username;
    }
    public String getPassword() {
        return password;
    }
    public void setPassword(String password) {
        this.password = password;
    }
    public String getBasePath() {
        return basePath;
    }
    public void setBasePath(String basePath) {
        this.basePath = basePath;
    }
    @Override
    public String toString() {
        return "FtpUtil [host=" + host + ", port=" + port + ", username=" + username + ", password=" + password
                + ", basePath=" + basePath + "]";
    }

}
具体使用
第一步：配置spring加载properties文件
applicationContext.xml
<context:property-placeholder location="classpath:*.properties"/>
ftp.properties
ftp.host=192.168.100.77
ftp.port=21
ftp.username=ftpuser
ftp.password=ftp54321
ftp.basePath=/home/ftpuser/
第二步：将工具类声明为bean
xml方式
<bean id="ftpUtil" class="com.cky.util.FtpUtil">
<property name="host" value="${ftp.host}"></property>
<property name="port" value="${ftp.port}"></property>
<property name="username" value="${ftp.username}"></property>
<property name="password" value="${ftp.password}"></property>
<property name="basePath" value="${ftp.basePath}"></property>
</bean>
注解方式，组件扫描
<context:component-scan base-package="com.cky.util"></context:component-scan>
第三步：注入使用
@Autowired
private FtpUtil ftpUtil;
FTP常见问题：
1、连接失败
(1)检查ftp服务器是否打开
(2)检查防火墙是否将ftp服务器端口加入白名单（测试的话也可以直接把防火墙关掉）
2、创建文件或者上传文件失败：
最常见的就是文件权限问题造成的，不光是读写权限，还有文件的用户组。
例如：ftpClient登录的用户是ftpUser   一般是文件存放在/home/ftpUser中，一般我们不会直接放在根目录，而是创建对应文件夹，这时需要注意，创建文件夹时需要以ftpUser或者同一组用户的身份创建，如果你是以root用户创建的，那么ftpUser将无权访问!
另