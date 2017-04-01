---
title: 在前端确认框上套用Promise
desc: jquery + bootstrap环境
author: ngtmuzi
category: 神秘代码
date: 2017-04-01 15:08:32
tags:
- Promise
- javaScript
- 前端
---
```javascript
//jquery + bootstrap
function askDlg(title, content) {
  return new Promise(function (resolve, reject) {
    $('#ask-title').text(title || '确认操作');
    $('#ask-content').text(content || '确认操作吗？');

    $('#ask-btn-yes').one('click', function () {
      resolve();
      $('#ask-dlg').modal('hide');
    });

    $('#ask-dlg').modal('show').one('hidden.bs.modal', reject);
  });
}
```