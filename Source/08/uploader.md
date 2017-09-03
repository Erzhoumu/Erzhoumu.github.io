---
title: uploader（多文件上传）
date: 2017-08-14
tags: web, tools
---
+ 描述：
    + 做单文件上传时，发现重新添加后，input值不发生改变，导致不知道是否成功了，尤其是文件很小的时候，进度条基本看不出来；
    + 查看源代码，发现dmuploader为多文件上传，所以会将文件名置“”也无可厚非；
    + 修改源代码之后，可现实当前文件名称；
    + 既然是多文件上传的源代码，索性加工一下，实现了一个多文件上传的工具
+ 效果截图：
+ js文件
    ``` js
/**
 * 多文件上传
 * @author zhangbo[zhangbo1@huli.com]
 * @since 2017-8-15
 */
(function($) {
    var pluginName = 'mulUploader';
    // These are the plugin defaults values
    var defaults = {
        url: document.URL,
        method: 'POST',
        extraData: {},
        maxFileSize: 0,
        maxFiles: 0,
        allowedTypes: '*',
        extFilter: null,
        dataType: null,
        fileName: 'file',
        onInit: function(){},
        onFallbackMode: function(message) {},
        onNewFile: function(id, file){},
        onBeforeUpload: function(index){},
        onComplete: function(file){},
        onUploadProgress: function(index, percent){},
        onUploadSuccess: function(index, data){},
        onUploadError: function(index, message){},
        onFileTypeError: function(file){},
        onFileSizeError: function(file){},
        onFileExtError: function(file){},
        onFilesMaxError: function(file){}
    };

    var MulUploader = function(element, options)
    {
        this.element = $(element);

        this.settings = $.extend({}, defaults, options);

        if(!this.checkBrowser()){
            return false;
        }

        this.init();

        return true;
    };

    MulUploader.prototype.checkBrowser = function()
    {
        if(window.FormData === undefined){
            this.settings.onFallbackMode.call(this.element, 'Browser doesn\'t support Form API');

            return false;
        }

        if(this.element.find('input[type=file]').length > 0){
            return true;
        }

        if (!this.checkEvent('drop', this.element) || !this.checkEvent('dragstart', this.element)){
            this.settings.onFallbackMode.call(this.element, 'Browser doesn\'t support Ajax Drag and Drop');

            return false;
        }

        return true;
    };

    MulUploader.prototype.checkEvent = function(eventName, element)
    {
        var element = element || document.createElement('div');
        var eventName = 'on' + eventName;

        var isSupported = eventName in element;

        if(!isSupported){
            if(!element.setAttribute){
                element = document.createElement('div');
            }
            if(element.setAttribute && element.removeAttribute){
                element.setAttribute(eventName, '');
                isSupported = typeof element[eventName] == 'function';

                if(typeof element[eventName] != 'undefined'){
                    element[eventName] = undefined;
                }
                element.removeAttribute(eventName);
            }
        }

        element = null;
        return isSupported;
    };

    MulUploader.prototype.init = function()
    {
        var widget = this;

        widget.queue = new Array();
        widget.queuePos = -1;
        widget.queueRunning = false;


        // -- Drag and drop event
        widget.element.on('drop', function (evt){
            evt.preventDefault();
            var files = evt.originalEvent.dataTransfer.files;
            widget.queueFiles(files);
        });


        //-- Optional File input to make a clickable area
        widget.element.find('input[type=file]').on('change', function(evt){
            var files = evt.target.files;
            widget.queueFiles(files);
        });

        this.settings.onInit.call(this.element);
    };

    MulUploader.prototype.queueFiles = function(files)
    {
        var j = this.queue.length;

        for (var i= 0; i < files.length; i++)
        {
            var file = files[i];

            // Check file size
            if((this.settings.maxFileSize > 0) &&
                (file.size > this.settings.maxFileSize)){

                this.settings.onFileSizeError.call(this.element, file);

                continue;
            }

            // Check file type
            if((this.settings.allowedTypes != '*') &&
                !file.type.match(this.settings.allowedTypes)){

                this.settings.onFileTypeError.call(this.element, file);

                continue;
            }

            // Check file extension
            if(this.settings.extFilter != null){
                var extList = this.settings.extFilter.toLowerCase().split(';');

                var ext = file.name.toLowerCase().split('.').pop();

                if($.inArray(ext, extList) < 0){
                    this.settings.onFileExtError.call(this.element, file);

                    continue;
                }
            }

            // Check max files
            if(this.settings.maxFiles > 0) {
                if(this.queue.length >= this.settings.maxFiles) {
                    this.settings.onFilesMaxError.call(this.element, file);

                    continue;
                }
            }

            this.queue.push(file);

            var index = this.queue.length - 1;

            this.settings.onNewFile.call(this.element, index, file);
        }

        // Only start Queue if we haven't!
        if(this.queueRunning){
            return false;
        }

        // and only if new Failes were successfully added
        if(this.queue.length == j){
            return false;
        }

        this.processQueue();

        return true;
    };

    MulUploader.prototype.processQueue = function()
    {
        var widget = this;

        widget.queuePos++;

        if(widget.queuePos >= widget.queue.length){
            // Cleanup

            widget.settings.onComplete.call(widget.element);

            // Wait until new files are droped
            widget.queuePos = (widget.queue.length - 1);

            widget.queueRunning = false;

            return;
        }

        var file = widget.queue[widget.queuePos];

        // Form Data
        var fd = new FormData();
        fd.append(widget.settings.fileName, file);

        // Return from client function (default === undefined)
        var can_continue = widget.settings.onBeforeUpload.call(widget.element, widget.queuePos);

        // If the client function doesn't return FALSE then continue
        if( false === can_continue ) {
            return;
        }

        // Append extra Form Data
        $.each(widget.settings.extraData, function(exKey, exVal){
            fd.append(exKey, exVal);
        });

        widget.queueRunning = true;

        // Ajax Submit
        $.ajax({
          url: widget.settings.url,
          type: widget.settings.method,
          dataType: widget.settings.dataType,
          data: fd,
          cache: false,
          contentType: false,
          processData: false,
          forceSync: false,
          xhr: function(){
            var xhrobj = $.ajaxSettings.xhr();
            if(xhrobj.upload){
              xhrobj.upload.addEventListener('progress', function(event) {
                var percent = 0;
                var position = event.loaded || event.position;
                var total = event.total || event.totalSize;
                if(event.lengthComputable){
                  percent = Math.ceil(position / total * 100);
                }
        
                widget.settings.onUploadProgress.call(widget.element, widget.queuePos, percent);
              }, false);
            }
        
            return xhrobj;
          },
          success: function (data, message, xhr){
            widget.settings.onUploadSuccess.call(widget.element, widget.queuePos, data);
          },
          error: function (xhr, status, errMsg){
            widget.settings.onUploadError.call(widget.element, widget.queuePos, errMsg);
          },
          complete: function(xhr, textStatus){
            widget.processQueue();
          }
        });
       // widget.settings.onUploadSuccess.call(widget.element, widget.queuePos, null);
        widget.processQueue();
    };

    $.fn.mulUploader = function(options){
        return this.each(function(){
            if(!$.data(this, pluginName)){
                $.data(this, pluginName, new MulUploader(this, options));
            }
        });
    };

    // -- Disable Document D&D events to prevent opening the file on browser when we drop them
    $(document).on('dragenter', function (e) { e.stopPropagation(); e.preventDefault(); });
    $(document).on('dragover', function (e) { e.stopPropagation(); e.preventDefault(); });
    $(document).on('drop', function (e) { e.stopPropagation(); e.preventDefault(); });
})(jQuery);
    ```
+ 使用
    + 页面元素
``` jsp
<div class="form-group">
    <label for="insCommitDoc" class="col-sm-4 control-label">上传文件：</label>
    <div class="col-sm-8" id="drag-and-drop-zone">
        <input type="file" id="insCommitDoc"  multiple="multiple" name="insCommitDoc"/>
        <div class="col-sm-8">
            <p id="uploadErrMsg" class="bg-danger">
            </p>
        </div>
    </div>
</div>
```
    + js绑定

``` js
/**
 * 多文件上传:支持Drop;多选;
 *
 */
var id = -1;
$('#drag-and-drop-zone').mulUploader({
    url: './upload',
    dataType: 'json',
    fileName: 'sfiles',
    allowedTypes: '*',
    extraData: {
        source: 1001,
        token: 123
    },
    maxFiles : 5,
    onInit:function() {
        //添加显示文件进度的DIV
        $(this).append("<div id='fileDiv' class='col-sm-8' style='max-height:300px;overflow-y: auto; margin-top: 10px;'></div>")
        id ++;
    },
    onUploadProgress: function (index, percent) {
        $("#"+index+"_progress" +" .progress-bar").css(
            'width',
            percent + '%'
        );
    },
    onUploadSuccess: function (index, data) {
        $.each(data, function (index, file) {
            if (file.statusCode == 200) {
                $("#" + id+"."+index).val(file.postfix);
                $("#" + index +"_progress" + " .progress-bar").css(
                    'width', '100%'
                ).html("上传完成");
            } else {
                console.log("statusCode:" + file.statusCode);
                $("#uploadErrMsg").html("statusCode:" + file.statusCode+" msg:"+file.errMsg);
            }
        });
    },
    onUploadError: function (id, message) {
        $("#uploadErrMsg").html(message);
    },
    onFileTypeError: function (file) {
        $("#uploadErrMsg").html(file.name + "cannot be added: must be an image");
    },
    onFileSizeError: function (file) {
        $("#uploadErrMsg").html(file.name + "cannot be added: size excess limit");
    },
    onFallbackMode: function (message) {
        $("#uploadErrMsg").html("Browser not supported(do something else here!): ");
    },
    onFileExtError: function(file){
        $("#uploadErrMsg").html(file.name + "cannot be added: extension excess error!");
    },
    onFilesMaxError: function(file){
        $("#uploadErrMsg").html(file.name + "cannot be added: num excess limit!");
    },
    onNewFile: function (index, file) {
        //新添加file
        $("#fileDiv").append("<div class='col-sm-8'>" +
                "<input type='text' value=" +file.name + ">" +
                "<input type='hidden' id="+id+"."+index +" name="+ id+"."+index+".name"+" value=''/>" +
                "<div id="+index+"_progress" + " class='progress'>" +
                    " <div class='progress-bar progress-bar-success progress-bar-striped' role='progressbar' style='width: 0%;'>"+
                    "</div>" +
                "</div>" +
            "</div>"
        );
    }
});
``` 




