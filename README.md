# HFUT-Student-QuickEval-One

## 功能简介

本脚本适用于合肥工业大学新版教务系统（eams5-student），可自动完成**当前学期下所有未完成的一位老师**的评教任务：

- 自动点击“未评教老师”
- 自动全选每题第一个选项
- 自动提交
- 支持检测 selectize 下拉框，支持你手动切换学期后自动再次运行
> ⚠️ **重要说明：**
> 
> 由于教务系统的机制，评教页面提交后会自动跳回默认学期（如2012-2013学年第一学期），脚本**不会自动切换学期**（之前还没有，之前会自动进入和跳转当前学期）。因此，本脚本的作用是一键完成“当前学期”下**一位未完成老师**的教评。如需评教其他老师，请手动切换学期后，脚本会自动再次批量评教（系统貌似做了些隐藏，个人水平有限没有实现自动跳转，等待大神实现）。

---

## 使用方法

1. **安装 [Tampermonkey](https://www.tampermonkey.net/) 浏览器插件**
2. **新建脚本**，点击添加新脚本，粘贴本仓库的 `user.js` 脚本内容并保存
   <img width="1028" height="740" alt="image" src="https://github.com/user-attachments/assets/f90359a1-77a4-4a48-8d51-f33691827f96" />
4. **打开评教首页**  
   <img width="2559" height="1570" alt="image" src="https://github.com/user-attachments/assets/adcc3b02-ea26-4208-bc2b-6291c6ffed32" />
5. **手动切换到你想要的学期**  
   脚本会自动完成该学期第一个未完成老师的评教。
6. **如需评教其他其他老师**  
   由于系统会自动跳回默认学期，请再次手动切换到目标学期，脚本会自动再次运行。
---

## 主要代码片段

```javascript
// ==UserScript==
// @name         评教单学期自动批量评教（模拟selectize点击）
// @namespace    http://tampermonkey.net/
// @version      1.1
// @description  自动完成当前学期所有老师的评教，支持模拟selectize人工点击切换学期
// @match        https://jxglstu.hfut.edu.cn/eams5-student/for-std/lesson-survey/semester-index/*
// @match        https://jxglstu.hfut.edu.cn/eams5-student/for-std/lesson-survey/start-survey/*
// @grant        none
// ==/UserScript==

(function() {
    'use strict';

    // ========== 模拟selectize人工点击切换学期 ==========
    function simulateSelectizeClick(targetSemesterText, callback) {
        // 1. 点击selectize的输入框，展开下拉
        let control = document.querySelector('.selectize-control');
        if (!control) return;
        control.click();

        // 2. 等待下拉菜单渲染
        setTimeout(() => {
            let options = document.querySelectorAll('.selectize-dropdown-content .option');
            let found = false;
            options.forEach(opt => {
                if (opt.textContent.trim() === targetSemesterText.trim()) {
                    opt.click();
                    found = true;
                }
            });
            if (found && typeof callback === 'function') {
                // 等待表格刷新
                waitForTableRefresh(callback);
            }
        }, 300);
    }

    function waitForTableRefresh(callback) {
        let table = document.querySelector('.table tbody');
        if (!table) {
            setTimeout(() => waitForTableRefresh(callback), 300);
            return;
        }
        let oldHtml = table.innerHTML;
        let observer = new MutationObserver(() => {
            if (table.innerHTML !== oldHtml) {
                observer.disconnect();
                setTimeout(callback, 500); // 等待内容稳定
            }
        });
        observer.observe(table, { childList: true, subtree: true });
    }

    // ========== 首页批量评教逻辑 ==========
    function batchEvaluateCurrentSemester() {
        setTimeout(() => {
            let links = Array.from(document.querySelectorAll('a[name="startSurvey"]'));
            if (links.length === 0) return;
            let i = 0;
            function clickNext() {
                if (i < links.length) {
                    links[i].click();
                    i++;
                    setTimeout(clickNext, 2000);
                }
            }
            clickNext();
        }, 1200);
    }

    // ========== 监听表格内容变化 ==========
    function observeTableChange() {
        let table = document.querySelector('.table tbody');
        if (!table) {
            setTimeout(observeTableChange, 500);
            return;
        }
        let lastHtml = table.innerHTML;
        let observer = new MutationObserver(() => {
            if (table.innerHTML !== lastHtml) {
                lastHtml = table.innerHTML;
                batchEvaluateCurrentSemester();
            }
        });
        observer.observe(table, { childList: true, subtree: true });
        // 首次进入页面也自动执行一次
        batchEvaluateCurrentSemester();
    }

    // ========== 评教页面自动全选并提交 ==========
    function surveyMain() {
        function selectFirstRadios() {
            let radios = document.querySelectorAll("input[type='radio']");
            let grouped = {};
            radios.forEach(radio => {
                let qid = radio.getAttribute('questionid');
                if (qid && !grouped[qid]) {
                    radio.checked = true;
                    grouped[qid] = true;
                }
            });
        }
        function autoSubmit() {
            let btn = document.getElementById('save-button');
            if (btn) {
                btn.click();
            }
        }
        setTimeout(() => {
            selectFirstRadios();
            setTimeout(autoSubmit, 1200);
        }, 1200);
    }

    // ========== 页面入口 ==========
    if (location.pathname.includes('/semester-index/')) {
        // 你可以在这里手动指定目标学期的文本，比如：
        // let targetSemesterText = "2023-2024学年第二学期";
        // simulateSelectizeClick(targetSemesterText, observeTableChange);

        // 或者只自动批量评教当前学期
        observeTableChange();
    } else if (location.pathname.includes('/start-survey/')) {
        window.addEventListener('load', function() {
            surveyMain();
        });
    }

    // ========== 提供一个全局函数，方便你在控制台手动调用 ==========
    window.simulateSelectizeClick = simulateSelectizeClick;
})();
```

---

## 注意事项

- 本脚本仅自动评教**当前学期下所有未完成老师**，如需评教其他学期，请手动切换学期。
- **评教页面提交后，系统会自动跳回默认学期（如2012-2013学年第一学期），需再次手动切换学期。**
- 评教页面会自动全选第一个选项并自动提交。
- 如遇页面结构变动或脚本失效，请提交 issue 或 PR。

---

## 免责声明

本脚本仅供学习与交流使用，请勿用于任何违反学校规定的用途。使用本脚本造成的任何后果由使用者本人承担。

---

## License

MIT 
