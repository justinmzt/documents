## canvas-直线拟合
### 建立画布

``` html
<canvas id="myCanvas" width="500" height="500" style="border:1px solid #000000;">
    您的浏览器不支持 HTML5 canvas 标签。
</canvas>
```
<canvas id="myCanvas" width="500" height="500" style="border:1px solid #000000;">
    您的浏览器不支持 HTML5 canvas 标签。
</canvas>

### 获取dom

``` javascript
    var canvas = document.getElementById("myCanvas");
    var context = canvas.getContext("2d");
```

### 获取点位置
注：由于画布的y坐标是从顶端开始计数，故y坐标在处理的时候做了`canvas.height - y`处理，后不做解释。

``` javascript
    function getLocation(x, y) {
        var bbox = canvas.getBoundingClientRect();
        return {
            x: (x - bbox.left) * (canvas.width / bbox.width),
            y: canvas.height - (y - bbox.top) * (canvas.height / bbox.height)
        };
    }
```

### 画点
这里画点用半径为一个像素的圆代替。
``` javascript
    function drawDot(x, y) {
        context.beginPath();
        context.arc(x, canvas.height - y, 1, 0, 2 * Math.PI);
        context.stroke();
        context.closePath();
    }
```

### 画线
这里画线`y = k*x + b`

``` javascript
    function drawLine(k, b) {
        context.beginPath();
        if(b < 0) context.moveTo(-b / k, canvas.height);
        else context.moveTo(0, canvas.height - b);
        context.lineTo(canvas.width, canvas.height - (canvas.width * k + b));
        context.stroke();
        context.closePath();
    }
```

### 实时获取坐标

``` html
<div>
    <p id="x"></p><p id="y"></p>
</div>
```
``` javascript
    canvas.onmousemove = function (e) {
        var location = getLocation(e.clientX, e.clientY);
        $("#x").text(location.x);//jQuery
        $("#y").text(location.y);
    };
```
### 拟合直线
根据最小二乘法推导出的公式：
<img src="http://img.my.csdn.net/uploads/201212/02/1354428824_3244.jpg">

``` javascript
    var samples = []; //所有点存在数组中
    canvas.onmouseup = function(e){
        var sum_x = 0;
        var sum_y = 0;
        var sum_xy = 0;
        var sum_x2 = 0;
        context.clearRect(0,0,canvas.width,canvas.height);//清除画布
        var location = getLocation(e.clientX, e.clientY);
        samples.push(location);
        var length = samples.length;
        for (var i in samples) {
            drawDot(samples[i].x, samples[i].y);
            sum_x += samples[i].x;
            sum_y += samples[i].y;
            sum_xy += samples[i].x * samples[i].y;
            sum_x2 += samples[i].x * samples[i].x;
        }
        var k = (length * sum_xy - sum_x * sum_y) / (length * sum_x2 - sum_x * sum_x);
        var b = (sum_y - sum_x * k) / length;
        drawLine(k, b)
    };
```
### 未完善之处

 - 没有异常点检测、异常点忽略等功能
 - 仅仅是线性回归的最初步的实践


  [1]: http://120.25.77.40/line.html