# websocket
websocket(简易聊天室)

/**
 * 
 * @authors wuchangjiang (qianduanwcj@126.com)

   The last edit
   
 * @date    2015-04-20 10:04:38
 * @version v2013
 */

var http=require('http');
var fs=require('fs');
var io=require('socket.io');
var mysql=require('mysql');
var httpServer=http.createServer(function (request, response){
	fs.readFile('www'+request.url, function (err, data){
		if(err){
			response.write('404');
		}else{
			response.write(data);
		}
		response.end();
	});
});
httpServer.listen(81);

//mysql
var db=mysql.createConnection({host: 'localhost', user: 'admin', password: '', database: 'student'});
function query(sql,fn){
	db.query(sql,fn);
}
//websocket
var socket=io.listen(httpServer);
var num=[];
socket.on('connection',function(sock){
	//在线状态以及人数
	num.push(sock);
	sock.on('disconnect',function(){
	//	console.log('有人断了');
		for(var i=0; i<num.length; i++){
			if(num[i]==sock){
				num.splice(i,1);
				break;
			}
		}
		sock.emit('line',num.length);
	});
	setInterval(function(){
		sock.emit('line',num.length);
	},1000)
	
	
	//注册 
	sock.on('reg',function(name,pass){
		query("SELECT * FROM user_name WHERE username='"+name+"'",function(err,data){
		console.log(data);
			if(err){
				sock.emit('reg_result',false,'数据库有错');
			}else if(data.length>0){
				sock.emit('reg_result',false,'用户名已经存在');
			}else{
				query("INSERT INTO user_name VALUES('"+name+"', '"+pass+"')", function (err, data){
					if(err){
						sock.emit('reg_result',false,'数据库有错');
					}else{
						sock.emit('reg_result',true,"注册成功，可以登入");
					}
				})
			}
		});
	});
	
	//登入
	sock.on('login',function(name,pass){
		query("SELECT * FROM user_name WHERE username='"+name+"'",function(err,data){
		console.log(data);
			if(err){
				sock.emit('login_result',false,'数据库错误')
			}else if(data.length==0){
				sock.emit('login_rasult',false,'账号不存在，可以注册');
			}else if(data[0].password!=pass){
				sock.emit('login_rasult',false,'密码错误');
			}else{
				sock.emit('login_rasult',true,'登录成功');
				
				//接收消息 发送消息
				sock.on('msg', function(str){
					for(var i=0; i<num.length; i++){
						if(num[i]==sock)continue;
						num[i].emit('recv_msg',str);
					}
					sock.emit('msg_result', true, '');
				});
			}
			
		});
	})
});
