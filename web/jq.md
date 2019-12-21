# web

这世间,还是绕不开前端呀.

---

### jquery 选择器

[选择器 link](https://runoob.com/cssref/css-selectors.html)

| 选择器          | 实例              | 选取                                      |
| --------------- | ----------------- | ----------------------------------------- |
| #id             | \$("#lastname")   | id="lastname" 的元素                      |
| .class          | \$(".intro")      | 所有 class="intro" 的元素                 |
| .class.class    | \$(".intro.demo") | 所有 class="intro" 且 class="demo" 的元素 |
| element         | \$("p")           | 所有 `<p>` 元素                           |
| element,element | div,p             | 选择所有`<div`>元素和`<p>`元素            |
| element element | div p             | 选择`<div`>元素内的所有`<p>`元素          |

---

### 动态绑定绑定事件

Q: 怎么对 jQuery 动态新增的元素进行绑定事件呢?

A: let me show you some black magic.

```html
<html>
  <head> </head>
  <body>
    <div id="app">
      <div class="class-1" data-id="1" data-name="haiyan">this is class 1</div>
    </div>

    <div id="test-area"></div>
  </body>

  <script src="https://cdn.staticfile.org/jquery/1.10.2/jquery.min.js"></script>
  <script>
    $(function() {
      // 动态新增元素绑定事件
      $("#test-area")
        .off("click", ".class-2")
        .on("click", ".class-2", function() {
          console.log("this is class 2");
        });

      // 这种绑定对动态新增元素无效.
      $(".class-2").on("click", function() {
        console.log("This is on .class-2");
      });

      $(".class-1").on("click", function() {
        var id = $(this).data("id");
        var name = $(this).data("name");
        console.log("id: " + id + " , name: " + name);

        var divHtml = "<div class='class-2'>this is class 2</div>";
        $("#test-area").append(divHtml);
      });
    });
  </script>
</html>
```

---

### radio 默认值

```html
<html>
  <head> </head>
  <body>
    <div id="app">
      <input type="radio" name="gender-radio" value="1" /> Male &nbsp;
      <input type="radio" name="gender-radio" value="0" /> Female

      <button id="confirm-btn">Ok</button>
      <button id="change-btn">Change</button>
    </div>
  </body>

  <script src="https://cdn.staticfile.org/jquery/1.10.2/jquery.min.js"></script>
  <script>
    // 在加载页面的时候,执行function(){}里面的东西
    $(function() {
      $("#confirm-btn").on("click", function() {
        var value = $("#app input[name='gender-radio']:checked").val();
        console.log(value);
      });

      $("#change-btn").on("click", function() {
        var value = $("#app input[name='gender-radio']:checked").val();
        if (value == 1) {
          $("#app input[name='gender-radio'][value='0']").prop(
            "checked",
            "checked"
          );
        } else {
          $("#app input[name='gender-radio'][value='1']").prop(
            "checked",
            "checked"
          );
        }

        console.log(value);
      });
    });
  </script>
</html>
```

---

### find 的使用

```html
<html>
  <head> </head>
  <body>
    <div id="app">
      <input type="text" name="search-name" placeholder="name" />
      <input type="text" name="search-age" placeholder="age" />

      <button id="get-it">Get it</button>
    </div>
  </body>

  <script src="https://cdn.staticfile.org/jquery/1.10.2/jquery.min.js"></script>
  <script>
    $(function() {
      $("#get-it").on("click", function() {
        var inputArr = $("#app").find("input");
        var values = new Array();

        inputArr.each(function() {
          var key = $(this).attr("name");
          var value = $(this).val();

          values.push({ key: key, value: value });
        });

        console.log(values);
      });
    });
  </script>
</html>
```
