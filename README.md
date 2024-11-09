# bullet_shooter
此项目为观众发送弹幕的客户端
部署前检查App.vue第31行：
``` javascript
socket = new WebSocket('ws://10.2.12.248:8080');
```
是否符合websocket服务器网址
运行 `npm run build` 编译静态网页
运行 `npm run start` 在本地部署网页