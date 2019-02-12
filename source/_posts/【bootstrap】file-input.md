---
title: 【bootstrap】file-input
date: 2018-11-05 15:11:52
tags:
  - bootstrap
  - fileinput
---

> 暂时不适用该方式上传文件

# 文件上传插件

官方文档：[fileinput](http://plugins.krajee.com/file-input)

# 使用

- 引入css
```html
<link rel="stylesheet" type="text/css" href="<c:url value="/assets/bower_components/bootstrap-fileinput/css/fileinput.min.css"/>"/>
```
- 引入js
```
<script type="text/javascript" src="<c:url value="/assets/bower_components/bootstrap-fileinput/js/fileinput.min.js"/>"></script>
<script type="text/javascript" src="<c:url value="/assets/bower_components/bootstrap-fileinput/js/locales/zh.js"/>"></script>
<script type="text/javascript" src="<c:url value="/assets/bower_components/bootstrap-fileinput/js/plugins/piexif.min.js"/>"></script>
```
该插件需要

```html
<input id="f1" type="file" name="file" class="file" data-show-preview="true"/>
```

```javascript

$scope.initFileinput("#f1");


$scope.initFileinput = function (fileinputId) {
    $(fileinputId).fileinput({
        "uploadUrl": "/fileUpload/upload",
        "maxFileSize": 20480,
        "msgSizeTooLarge": "文件({name} - {size} KB) 超过最大限制 {maxSize} KB. 请重新上传!"
    });
    $(fileinputId).on('fileuploaded', function(event, data, previewId, index) {
        $scope.taskRequest.fileName = data.response.fileName;
        $scope.taskRequest.fileUrl = data.response.path;
    });
};

```



