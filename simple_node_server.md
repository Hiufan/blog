# simple_node_server

1.install dependecies    
  `$ npm install connect serve-static`
  
2.create server.js
```javascript
  var connect = require('connect');
  var serveStatic = require('serve-static');
  connect().use(serveStatic(__dirname)).listen(8080);  
```

3.run server
```shell
node server.js
```
