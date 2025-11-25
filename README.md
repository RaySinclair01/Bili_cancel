# Bili_cancel
新版b站批量取关



代码如下：
```javascripts
// ==UserScript==
// @name         Bilibili 批量取关 (适配新版UI)
// @namespace    http://tampermonkey.net/
// @version      2.0
// @description  批量取消 B 站关注，适配新版 UI 和翻页
// @author       Modified
// @match        https://space.bilibili.com/*/relation/follow*
// @icon         https://www.bilibili.com/favicon.ico
// @grant        none
// @require      https://code.jquery.com/jquery-3.6.0.min.js
// ==/UserScript==

(function() {
    'use strict';
    const $ = window.jQuery;

    // === 配置区域 ===
    const CONFIG = {
        clickDelay: 800,      // 取关一个人的间隔 (毫秒)，太快会被B站风控
        pageDelay: 3000,      // 翻页后的等待时间 (毫秒)
        maxPages: 100         // 最大翻页数
    };

    // 添加控制按钮
    function addControlPanel() {
        if ($('#unfollow-btn-start').length > 0) return;

        const btn = $(`<button id="unfollow-btn-start" style="
            position: fixed;
            top: 100px;
            right: 20px;
            z-index: 9999;
            padding: 10px 20px;
            background: #FF6699;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
            font-weight: bold;
            font-size: 16px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.3);
        ">开始批量取关</button>`);

        $('body').append(btn);

        btn.click(async function() {
            if (!confirm('确定要开始批量取关吗？\n请保持网页在前台运行，不要最小化。')) return;
            $(this).text('正在运行...').prop('disabled', true).css('background', '#ccc');
            await startUnfollowProcess();
            $(this).text('任务结束').css('background', '#4CAF50');
        });
    }

    // 延时函数
    function sleep(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }

    // 处理单个页面的取关
    async function processCurrentPage() {
        // 1. 找到所有“已关注”的按钮 (灰色按钮)
        // 根据你提供的 HTML: <div class="follow-btn__trigger gray">
        const buttons = $(".follow-btn__trigger.gray");

        if (buttons.length === 0) {
            console.log("✅ 当前页没有检测到已关注的人，或者页面未加载完成。");
            return false;
        }

        console.log(`👉 当前页发现 ${buttons.length} 个关注对象`);

        for (let i = 0; i < buttons.length; i++) {
            const btn = $(buttons[i]);

            // 再次检查状态，防止重复点击
            if (!btn.hasClass('gray')) continue;

            // 点击“已关注”按钮
            btn.click();
            console.log(`正在取关第 ${i + 1} 个...`);

            await sleep(300); // 等待弹窗出现

            // --- 处理 B 站的二次确认弹窗 ---
            // B站经常会弹出一个 modal 问你是否确定，我们需要点击确定
            // 查找通用的确认按钮类名，通常是 modal 里的 primary button
            const confirmBtn = $(".vui_modal .vui_button--primary, .bili-modal-footer .primary");
            if (confirmBtn.length > 0 && confirmBtn.is(':visible')) {
                confirmBtn.click();
                console.log("  -> 点击了二次确认弹窗");
            }

            // 等待操作完成
            await sleep(CONFIG.clickDelay);
        }
        return true;
    }

    // 翻页逻辑
    async function goToNextPage() {
        // 根据你提供的 HTML: <button class="vui_button vui_pagenation--btn vui_pagenation--btn-side">下一页</button>
        // 我们使用 :contains 选择器来确保点到的是“下一页”而不是“上一页”
        const nextBtn = $("button.vui_pagenation--btn-side:contains('下一页')");

        if (nextBtn.length === 0) {
            console.log("🚫 未找到下一页按钮");
            return false;
        }

        if (nextBtn.prop('disabled') || nextBtn.hasClass('vui_button--disabled')) {
            console.log("🚫 下一页按钮不可点，已到达最后一页");
            return false;
        }

        console.log("➡️ 正在翻页...");
        nextBtn.click();
        return true;
    }

    // 主流程
    async function startUnfollowProcess() {
        let page = 1;
        while (page <= CONFIG.maxPages) {
            console.log(`📄 正在处理第 ${page} 页`);

            // 处理当前页
            await processCurrentPage();

            // 尝试翻页
            const hasNext = await goToNextPage();
            if (!hasNext) {
                console.log("🏁 任务全部完成！");
                break;
            }

            // 等待新页面加载
            await sleep(CONFIG.pageDelay);
            page++;
        }
    }

    // 脚本入口
    $(document).ready(() => {
        // 等待页面元素稍微加载一下再显示按钮
        setTimeout(addControlPanel, 2000);
    });

})();
```
