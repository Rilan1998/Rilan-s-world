demo源码  http套接字完整封装部分
=====================================
emmmmmm这部分代码量过于大，有一千行以上了，关键部分给了注释，其他一小点非关键部分有一些是借鉴成熟项目的部分可能会留一点坑，可能不再一一总结了，这部分工作流程不难，主要还是工具类函数很多很精简很厉害，这部分函数基本上都是在大神的代码里学到的，里面有一些c++的字符串处理的艺术，对我个人的技术成长有很大启发，真诚的再次感谢！
其他不在赘述，终于写完啦！虽然有些部分还有些缺憾，但是成就感满满，希望这种成就感可以让我在代码的道路上走的更远，而不是仅仅感动了当下的自己。

http.h
```
#ifndef _HTTP_H_
#define _HTTP_H_

#include <sys/types.h>
#include <map>
#include <string>
#include <stdio.h>
#include <string.h>
#include <sys/stat.h>
#include <libgen.h>
#include "socket.h"
#include "event.h"

namespace Http
{
#define HTTP_VERSION			"HTTP/1.1"

#define HTTP_HEAD_CONTENT_TYPE	"Content-type"
#define HTTP_HEAD_CONTENT_LEN	"Content-length"
#define HTTP_HEAD_CONNECTION	"Connection"
#define HTTP_HEAD_KEEP_ALIVE	"Keep-Alive"

#define HTTP_ATTR_KEEP_ALIVE 	"keep-alive"

#define HTTP_HEAD_HTML_TYPE		"text/html"
#define HTTP_HEAD_CSS_TYPE		"text/css"
#define HTTP_HEAD_GIF_TYPE		"image/gif"
#define HTTP_HEAD_JPG_TYPE		"image/jpeg"
#define HTTP_HEAD_PNG_TYPE		"image/png"

#define HTTP_ROOT "Html"
#define HTTP_DEFAULTFILE 	"index.html"
#define HTTP_SLASH 			"/"
#define HTTP_CURRENT_DIR	"."
#define HTTP_ABOVE_DIR 		".."

inline size_t addrlen(const char *start, const char *end);
std::string  extention_name(const std::string &basename);
const char *find_content(const char *start, const char *end, char endc, size_t &contlen, size_t &sumlen);
const char *find_line(const char *start, const char *end);
const char *find_headline(const char *start, const char *end);
const char *http_content_type(const std::string &extension);
typedef std::map<std::string, std::string> HttpHead_t;

std::string http_path_handle(const std::string &dname, const std::string &bname);
inline void split_url(const std::string &url, std::string &dir, std::string &base);

class  HttpRequest
{
	public:
		HttpRequest();
		~HttpRequest();
		const std::string &start_line();
		const std::string &method();
		const std::string &url();
		const std::string &version();
		const HttpHead_t &headers();
		const size_t body_len();
		const char *body();

		int load_packet(const char *msg, size_t msglen);
	private:
		int parse_startline(const char *start, const char *end);//处理开始行
		int parse_headers(const char *start, const char *end);
		int parse_body(const char *start, const char *end);
	private:
	/*实例
GET /hello.txt HTTP/1.1
User-Agent: curl/7.16.3 libcurl/7.16.3 OpenSSL/0.9.7l zlib/1.2.3
Host: www.example.com
Accept-Language: en, mi
	
m_startline(m_method  m_url  m_version )
m_headers 
m_body

	*/
		std::string m_startline;
		std::string m_method;
		std::string m_url;
		std::string m_version;
		HttpHead_t m_headers;
		char *m_body;
		size_t m_bodylen;
};

class HttpRespose
{
	public:
		HttpRespose();
		~HttpRespose();
		int set_version(const std::string &version);
		int set_status(const std::string &status, const std::string &reason);
		int add_head(const std::string &name, const std::string &attr);
		int del_head(const std::string &name);
		int set_body(const char *body, size_t bodylen);
		size_t size();
		const char *serialize();
	private:
		size_t startline_stringsize();
		size_t headers_stringsize();
	private:
		enum Config{
			MAXLINE = 1024,
			BODY_MAXSIZE = 64 * 1024,
		};

/*响应消息实例
HTTP/1.1 200 OK 状态行
Date: Mon, 27 Jul 2009 12:28:53 GMT
Server: Apache
Last-Modified: Wed, 22 Jul 2009 19:15:56 GMT
ETag: "34aa387-d-1568eb00"
Accept-Ranges: bytes
Content-Length: 51
Vary: Accept-Encoding
Content-Type: text/plain   消息报头
                                                          空行
***************************  html响应正文		
*/
		typedef struct {
			std::string version;
			std::string status;
			std::string reason;
			HttpHead_t headers;//消息报头
			char *body;
			size_t bodylen;
			char *data;
			size_t datalen;
			size_t totalsize;
			bool dirty;//是否修改过
		}respose_package_t;
	private:
		respose_package_t m_package;
};




class HttpStream: public Event::EventHandle
{
	public:
		HttpStream(Socket::Client *);
		~HttpStream();
		int close();
		void handle_in(int fd);
		void handle_close(int fd);
	private:
		Http::HttpRespose *handle_request(Http::HttpRequest &request);
	private:
		enum CONFIG{
			READBUFF_LEN = 1024,
		};
	private:
		Socket::Client *m_client;
		pthread_mutex_t m_readbuffmutex;
		char *m_readbuff;
};


class HttpServer :public Event::EventHandle
{
	public:
		HttpServer(const std::string &addr, uint16_t port);
		~HttpServer();
		int start(size_t backlog);
		int close();
	protected:
		void handle_in(int fd);
	private:
		pthread_mutex_t m_sockmutex;
		uint16_t m_port;
		const std::string m_addr;
		Socket::Server m_server;
};

}

#endif
```
http.cpp
```
#include"http.h"




namespace Http
{
size_t addrlen(const char *start, const char *end)
{
	return (size_t)(end - start);
}
std::string  extention_name(const std::string &basename)
{
	std::string extention("");
	size_t pos = basename.find_last_of('.');
	if(pos == std::string::npos)
		return extention;
	return basename.substr(pos + 1);
}

const char *find_content(const char *start, const char *end, char endc, size_t &contlen, size_t &sumlen)
{
	size_t contentlen = 0;
	const char *contstart = NULL;//指向字符常量的指针
	for(const char *mstart = start; mstart < end; mstart++){
		if(contstart == NULL){
			if(*mstart != ' '){//最开始时
				contstart = mstart;
				contentlen = 1;
			}
		}else{
			if(*mstart == endc){//找到时，结束
				contlen = contentlen;//关键词长度
				sumlen = addrlen(start, mstart);//遍历过的长度
				return contstart;//返回关键词的起始字符
			}
			contentlen++;
		}
	}

	return NULL;
}

const char * find_line(const char *start, const char *end)
{
	for(const char *lstart = start; lstart < (end - 1); lstart++){
		if(lstart[0] == '\r' && lstart[1] == '\n'){
			return &lstart[2];
		}
	}
	return NULL;
}
const char * find_headline(const char *start, const char *end)
{
	for(const char *hstart = start; hstart < (end - 3); hstart++){
		if(hstart[0] == '\r' && hstart[1] == '\n' && hstart[2] == '\r' && hstart[3] == '\n'){
			return &hstart[4];
		}
	}

	return NULL;
}
const char *http_content_type(const std::string &extension)
{
	if(extension.compare("html") == 0x00)
		return HTTP_HEAD_HTML_TYPE;
	else if(extension.compare("css") == 0x00)
		return HTTP_HEAD_CSS_TYPE;
	else if(extension.compare("gif") == 0x00)
		return HTTP_HEAD_GIF_TYPE;
	else if(extension.compare("jpg") == 0x00)
		return HTTP_HEAD_JPG_TYPE;
	else if(extension.compare("png") == 0x00)
		return HTTP_HEAD_PNG_TYPE;
	return NULL;
}

std::string http_path_handle(const std::string &dname, const std::string &bname)
{
	std::string filepath(HTTP_ROOT);
	if(bname == HTTP_SLASH || bname == HTTP_CURRENT_DIR || \
		bname == HTTP_ABOVE_DIR){	
		filepath += HTTP_SLASH;
		filepath += HTTP_DEFAULTFILE;
	}else if(dname == HTTP_CURRENT_DIR){
		filepath += HTTP_SLASH;
		filepath += bname;
	}else if(dname == HTTP_SLASH){
		filepath += dname;
		filepath += bname;
	}else{
		filepath += dname;
		filepath += HTTP_SLASH;
		filepath += bname;
	}

	return filepath;
}
void split_url(const std::string &url, std::string &dir, std::string &base)
{
	char *dirc = strdup(url.c_str());
	dir = dirname(dirc);
	delete dirc;
	
	char *basec = strdup(url.c_str());
	base = basename(basec);	
	delete basec;	
}

//构造与析构
HttpRequest::HttpRequest():m_body(NULL), m_bodylen(0)
{
}
HttpRequest::~HttpRequest()
{
	if(m_body != NULL)
		delete m_body;
}



//const get类函数
const std::string &HttpRequest::start_line()
{
	return m_startline;
}
const std::string &HttpRequest::method()
{
	return m_method;
}
const std::string &HttpRequest::url()
{
	return m_url;
}
const std::string &HttpRequest::version()
{
	return m_version;
}
const HttpHead_t &HttpRequest::headers()
{
	return m_headers;
}
const size_t HttpRequest::body_len()
{
	return m_bodylen;
}
const char *HttpRequest::body()
{
	return m_body;
}


//处理网络包，用于handin函数中，包含处理开始行，请求头，体三个函数。
int HttpRequest::load_packet(const char *msg, size_t msglen)
{
	const char *remainmsg = msg;
	const char *endmsg = msg + msglen;
	const char *endline = find_line(remainmsg, endmsg);	//parse start line

	parse_startline(remainmsg, endline);

	remainmsg = endline;
	const char *headline_end = find_headline(remainmsg, endmsg);

	parse_headers(remainmsg, endmsg) ;

	remainmsg = headline_end;
	parse_body(remainmsg, endmsg) ;

	return 0;
}

//处理开始行
int HttpRequest::parse_startline(const char *start, const char *end)
{
	size_t contlen = 0, sumlen = 0;
	const char *cont = NULL, *remainbuff = start;

	cont = find_content(remainbuff, end, '\r', contlen, sumlen);
	m_startline = std::string(cont, contlen);
	
	cont = find_content(remainbuff, end, ' ', contlen, sumlen);
	m_method = std::string(cont, contlen);
	remainbuff += sumlen;

	cont = find_content(remainbuff, end, ' ', contlen, sumlen);
	m_url = std::string(cont, contlen);
	remainbuff += sumlen;

	cont = find_content(remainbuff, end, '\r', contlen, sumlen);
	m_version = std::string(cont, contlen);	
	return 0;
}
//处理请求头
int HttpRequest::parse_headers(const char *start, const char *end)
{	
	size_t contlen = 0, sumlen = 0;
	const char *line_start = start;
	std::string head, attr;
	m_headers.clear();
	for(;;){
		const char *line_end = find_line(line_start, end);
		if(line_end == NULL)
			return -1;
		else if(line_end == end)	// end
			break;

		const char *headstart = find_content(line_start, line_end, ':', contlen, sumlen);
		if(headstart == NULL)
			return -1;
		head = std::string(headstart, contlen);

		const char *attrstart = line_start + sumlen + 0x01;
		attrstart = find_content(attrstart, line_end, '\r', contlen, sumlen);
		if(attrstart == NULL)
			return -1;
		attr = std::string(attrstart, contlen);

		line_start = line_end;
		m_headers[head] = attr;
	}

	return 0;
}
//处理请求体
int HttpRequest::parse_body(const char *start, const char *end)
{
	size_t bodylen = addrlen(start, end);
	if(bodylen == 0x00)
		return 0;

	char *buff = new char[bodylen];
	memcpy(buff, start, bodylen);
	
	if(m_body != NULL)
		delete m_body;
	m_body = buff;
	m_bodylen = bodylen;
	return 0;
}





//构造与析构
HttpRespose::HttpRespose()
{
	m_package.body = NULL;
	m_package.bodylen = 0x00;
	m_package.data = NULL;
	m_package.datalen = 0x00;
	m_package.dirty = true;
}
HttpRespose::~HttpRespose()
{
	if(m_package.body != NULL)
		delete [] m_package.body;
	if(m_package.data != NULL)
		delete [] m_package.data;
}



int HttpRespose::set_version(const std::string &version)
{
	m_package.version = version;
	m_package.dirty = true;
	return 0;
}
int HttpRespose::set_status(const std::string &status, const std::string &reason)
{
	m_package.status = status;
	m_package.reason = reason;
	m_package.dirty = true;
	return 0;
}
int HttpRespose::add_head(const std::string &name, const std::string &attr)
{
	if(name.empty() || attr.empty())
		return -1;

	m_package.headers[name] = attr;
	m_package.dirty = true;
	return 0;
}
int HttpRespose::del_head(const std::string &name)
{
	HttpHead_t::iterator itor = m_package.headers.find(name);
	if(itor == m_package.headers.end())
		return -1;
	m_package.headers.erase(itor);
	m_package.dirty = true;
	return 0;
}
int HttpRespose::set_body(const char *body, size_t bodylen)
{
	if(body == NULL || bodylen == 0x00 || bodylen > BODY_MAXSIZE)
		return -1;

	char *buff = new char[bodylen];
	assert(buff != NULL);
	
	memcpy(buff, body, bodylen);

	if(m_package.body != NULL)
		delete [] m_package.body;
	m_package.body = buff;
	m_package.bodylen = bodylen;
	m_package.dirty = true;
	return 0;
}
size_t HttpRespose::size()
{
	if(m_package.dirty){
		m_package.totalsize = startline_stringsize() + headers_stringsize();
		m_package.totalsize += m_package.bodylen;
	}
	return m_package.totalsize;
}


const char *HttpRespose::serialize()//将表单内容序列化成一个字符串
{
	if(!m_package.dirty)
		return m_package.data;

	size_t totalsize = size();
	char *buffreserver = new char[totalsize];
	assert(buffreserver != NULL);

	char *buff = buffreserver;
	int nprint = snprintf(buff, totalsize, "%s %s %s\r\n", m_package.version.c_str(), \
		m_package.status.c_str(), m_package.reason.c_str());
	if(nprint < 0)
		goto ErrorHandle;

	totalsize -= nprint;
	buff += nprint;

	for(HttpHead_t::iterator itor = m_package.headers.begin(); itor != m_package.headers.end(); itor++){
		const std::string &name = itor->first;
		const std::string &attr = itor->second;

		nprint = snprintf(buff, totalsize, "%s: %s\r\n", name.c_str(), attr.c_str());
		if(nprint < 0)
			goto ErrorHandle;
		totalsize -= nprint;
		buff += nprint;
	}	

	nprint = snprintf(buff, totalsize, "\r\n");
	if(nprint < 0)
		goto ErrorHandle;
	totalsize -= nprint;
	buff += nprint;
	
	memcpy(buff, m_package.body, totalsize);
	if(m_package.data != NULL)
		delete m_package.data;
	m_package.data = buffreserver;

	m_package.dirty = false;
	return m_package.data;
ErrorHandle:
	delete buffreserver;
	return NULL;
}
size_t HttpRespose::startline_stringsize()
{
	const size_t otherchar_size = 1 * 2 + 2;  // black * 2 + CRLF
	size_t total_size = otherchar_size + m_package.version.size();
	total_size += m_package.status.size()+ m_package.reason.size();
	return total_size;
}
size_t HttpRespose::headers_stringsize()
{
	const size_t otherchar_size = 2 + 2;	// ': ' + CRLF
	const size_t head_terminatorsize = 2;	// CRLF

	size_t stringsize = 0;
	HttpHead_t::iterator itor = m_package.headers.begin();
	for(; itor != m_package.headers.end(); itor++){
		const std::string &name = itor->first;
		const std::string &attr = itor->second;

		stringsize += name.size() + attr.size() + otherchar_size;
	}

	stringsize += head_terminatorsize;
	return stringsize;
}





HttpStream::HttpStream(Socket::Client *iclient): m_client(iclient)
{
	m_readbuff = new char[READBUFF_LEN + 1];
	assert(m_readbuff != NULL);

	m_client->set_nonblock(true);

	/*
	* 构造最后注册进入epoll
	* 防止未构造完全收到事件回调
	*/
	register_event(*m_client);
}


HttpStream::~HttpStream()
{
	delete m_client;
	m_client = NULL;
	
	delete m_readbuff;
	m_readbuff = NULL;
}
int HttpStream::close()
{
	return shutdown_event(*m_client);
}


void HttpStream::handle_in(int fd)
{
	pthread_mutex_lock(&m_readbuffmutex);
	ssize_t nread = m_client->recv(m_readbuff, READBUFF_LEN, MSG_DONTWAIT);
	if((nread < 0 && nread != EAGAIN) || nread == 0){	// error or read EOF
		close();
		return;
	}else if(nread < 0 && nread == EAGAIN){ 			
		printf("non data to read\n");
		return; 
	}	
	m_readbuff[nread] = 0x00;

	Http::HttpRequest httprequest;
	httprequest.load_packet(m_readbuff, nread) ;


	printf("socket[%d] receive <--- %ld bytes\n", fd, nread);
	Http::HttpRespose *respose = handle_request(httprequest);
	if(respose != NULL){
		m_client->send(respose->serialize(), respose->size(), 0);
		delete respose;
	}
}
void HttpStream::handle_close(int fd)
{
	delete this;
}

Http::HttpRespose *HttpStream::handle_request(Http::HttpRequest &request)
{
	const std::string &method = request.method();
	const std::string &url = request.url();

	std::string dname, bname;
	split_url(url, dname, bname);//url分解为目录名和文件名

	Http::HttpRespose *respose = new Http::HttpRespose;	
	std::string filepath = http_path_handle(dname, bname);//对文件路径所在不同情况进行判断
	if(method == "GET"){

		std::string extention = extention_name(filepath);//后缀名
		if(extention.empty() || access(filepath.c_str(), R_OK) < 0){//判断权限

			printf("access %s error\n", filepath.c_str());
			
			respose->set_version(HTTP_VERSION);
			respose->set_status("404", "Not Found");
			respose->add_head(HTTP_HEAD_CONNECTION, "close");
			return respose;
		}

		
		struct stat filestat;
		stat(filepath.c_str() ,  &filestat);
		const size_t filesize = filestat.st_size;

		char *fbuff = new char[filesize];
		assert(fbuff != NULL);

		FILE *fp = fopen(filepath.c_str(), "rb");
		if(fp == NULL || fread(fbuff, filesize, 1, fp) != 1){//打开失败，返回500
		
			delete fbuff;

			respose->set_version(HTTP_VERSION);
			respose->set_status("500", "Internal Server Error");
			respose->add_head(HTTP_HEAD_CONNECTION, "close");
			return respose;
		}
		
		fclose(fp);

		char sfilesize[16] = {0x00};
		snprintf(sfilesize, sizeof(sfilesize), "%ld", filesize);

		respose->set_version(HTTP_VERSION);
		respose->set_status("200", "OK");//打开成功 返回200
		respose->add_head(HTTP_HEAD_CONTENT_TYPE, http_content_type(extention));
		respose->add_head(HTTP_HEAD_CONTENT_LEN, sfilesize);
		respose->add_head(HTTP_HEAD_CONNECTION, "close");
		respose->set_body(fbuff, filesize);
		delete fbuff;
	}

	return respose;
}




HttpServer::HttpServer(const std::string &addr, uint16_t port)
:
	m_port(port), m_addr(addr), m_server(addr, port)
{
	printf("constructor ip %s, port %d\n", addr.c_str(), port);
}

int HttpServer::start(size_t backlog)
{
	if(!m_server.isclose())	 //has start
		return 0;
		
	m_server.start(backlog) ;
	m_server.set_nonblock(true);

	return register_event(m_server);
}

void HttpServer::handle_in(int fd)
{

	/*
	* 读取所有建立的连接
	*/
	do{
		Socket::Client *newconn = m_server.accept();
		if(newconn == NULL){
			if(errno == EAGAIN || errno == EWOULDBLOCK || errno == EINTR){
				break;
			}else{
				printf("sockfd[%d] accept error\n", fd);
				close();
				return;
			}
		}
		
		printf("socket[%d] accept a conn\n", fd);
		HttpStream *httpstream = new HttpStream(newconn);	// self release 
		assert(httpstream != NULL);

	}while(true);



}


}
```

