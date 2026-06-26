# bili-feed-filter-script
👇AI油猴脚本
```javascript
// ==UserScript==
// @name         Bot站首页视频过滤
// @namespace    http://tampermonkey.net/
// @version      1.0
// @description  先走白名单绝对放行，再走黑名单精准屏蔽，数据持久化存储，支持一键重置，支持多维度过滤(全部/视频/UP主)
// @author       FuckAIAccount
// @match        https://www.bilibili.com/
// @match        https://www.bilibili.com/?*
// @icon         https://www.bilibili.com/favicon.ico
// @grant        GM_setValue
// @grant        GM_getValue
// @run-at       document-end
// ==/UserScript==

(function() {
    'use strict';

    // 规则配置项说明：
    // type:   'keyword' (文本关键字) / 'regexp' (正则表达式)
    // filter: 'all' (同时匹配标题和UP主) / 'video' (仅匹配视频标题) / 'up' (仅匹配UP主名称)
    const defaultRules = {
        white: [
        ],
        black: [
            { type: 'keyword', filter: 'up', value: '纪录片' },
            { type: 'keyword', filter: 'up', value: '大型' },
            { type: 'regexp', filter: 'up', value: '动漫$' },
            { type: 'keyword', filter: 'up', value: '漫剪' },
            { type: 'keyword', filter: 'up', value: '话题' },
            { type: 'regexp', filter: 'up', value: '影视$' },
            { type: 'keyword', filter: 'up', value: '每日' },
            { type: 'regexp', filter: 'up', value: '分享$' },
            { type: 'keyword', filter: 'up', value: '推荐' },
            { type: 'regexp', filter: 'up', value: '^bili_' },
            { type: 'regexp', filter: 'up', value: '解说$' },
            { type: 'keyword', filter: 'up', value: '剧场' },
            { type: 'keyword', filter: 'up', value: '情报局' },
            { type: 'keyword', filter: 'up', value: '情报员' },
            { type: 'keyword', filter: 'up', value: '观察局' },
            { type: 'keyword', filter: 'up', value: '杂谈' },
            { type: 'keyword', filter: 'up', value: '昵称' },
            { type: 'keyword', filter: 'up', value: '官方频道' },
            { type: 'keyword', filter: 'up', value: '官方资源' },
            { type: 'regexp', filter: 'up', value: '观察$' },
            { type: 'regexp', filter: 'up', value: '^电影' },
            { type: 'keyword', filter: 'up', value: '大师' },
            { type: 'regexp', filter: 'up', value: '科普$' },
            { type: 'regexp', filter: 'up', value: '解读$' },
            { type: 'keyword', filter: 'up', value: '网友' },
            { type: 'keyword', filter: 'video', value: '大型纪录片' },
            { type: 'regexp', filter: 'video', value: '^男子' },
            { type: 'keyword', filter: 'video', value: '为什么不' },
            { type: 'keyword', filter: 'video', value: '有哪些' },
            { type: 'regexp', filter: 'video', value: '^为啥' },
            { type: 'regexp', filter: 'video', value: '为什么.*我们' },
            { type: 'keyword', filter: 'video', value: '：为什么' },
            { type: 'regexp', filter: 'video', value: '^为什么说' },
            { type: 'regexp', filter: 'video', value: '^(为何|为啥|为什么|为什么现在|为什么我|你觉得|到底是|你听过|你见过|你们|为什么很多人)' },
            { type: 'regexp', filter: 'video', value: '.+为何.+？$'},
            { type: 'regexp', filter: 'video', value: '^如何看待' },
            { type: 'regexp', filter: 'video', value: '^如何评价' },
            { type: 'regexp', filter: 'video', value: '^如何反驳' },
            { type: 'regexp', filter: 'video', value: '是什么？$' },
            { type: 'regexp', filter: 'video', value: '怎么看？$' },
            { type: 'regexp', filter: 'video', value: '发生吗？$' },
            { type: 'regexp', filter: 'video', value: '(!|?)$' },
            { type: 'regexp', filter: 'video', value: '^哪些事情' },
            { type: 'regexp', filter: 'video', value: '^普通人' },
            { type: 'regexp', filter: 'video', value: '真的是.+？' },
            { type: 'regexp', filter: 'video', value: '反应.+？' },
            { type: 'regexp', filter: 'video', value: '的容忍度$' },
            { type: 'keyword', filter: 'video', value: '分享经验' },
            { type: 'regexp', filter: 'video', value: '普通人学.+？' },
            { type: 'regexp', filter: 'video', value: '普通人自学.+！' },
            { type: 'keyword', filter: 'video', value: '骂醒一个' },
            { type: 'regexp', filter: 'video', value: '^终于明白' },
            { type: 'regexp', filter: 'video', value: '发展史！$' },
            { type: 'regexp', filter: 'video', value: '^【中(字|配)】' },
            { type: 'regexp', filter: 'video', value: '^我的世界：' },
            { type: 'keyword', filter: 'all', value: '鸿蒙' },
            { type: 'keyword', filter: 'all', value: '华为' },
            { type: 'keyword', filter: 'all', value: '问界' },
            { type: 'keyword', filter: 'all', value: '智界' },
            { type: 'keyword', filter: 'all', value: '享界' },
            { type: 'keyword', filter: 'all', value: '尚界' },
            { type: 'keyword', filter: 'all', value: '尊界' },
            { type: 'keyword', filter: 'all', value: '乾崑' },
            { type: 'keyword', filter: 'all', value: '阿维塔' },
        ]
    };

    let savedRules = GM_getValue('filter_rules', JSON.parse(JSON.stringify(defaultRules)));

    // 编译后的规则数组
    let whiteCompiled = [];
    let blackCompiled = [];

    // 解析并编译规则
    function compileRules() {
        const parseRule = r => {
            try {
                return {
                    filter: r.filter || 'all',
                    reg: new RegExp(r.value)
                };
            } catch(e) {
                return null;
            }
        };
        whiteCompiled = savedRules.white.map(parseRule).filter(Boolean);
        blackCompiled = savedRules.black.map(parseRule).filter(Boolean);
    }
    compileRules();

    let toastTimer = null;

    // 显示提示消息
    function showToast(message) {
        let toast = document.getElementById('filter-toast');
        if (!toast) {
            toast = document.createElement('div');
            toast.id = 'filter-toast';
            toast.style.cssText = `
                position: fixed; bottom: 20px; right: 20px; background: rgba(0, 0, 0, 0.8);
                color: #fff; padding: 10px 16px; border-radius: 4px; z-index: 99999;
                font-size: 14px; box-shadow: 0 2px 8px rgba(0,0,0,0.2); transition: opacity 0.3s ease; pointer-events: none;
            `;
            document.body.appendChild(toast);
        }
        toast.textContent = message;
        toast.style.opacity = '1';
        clearTimeout(toastTimer);
        toastTimer = setTimeout(() => { toast.style.opacity = '0'; }, 3000);
    }

    // 校验文本是否命中规则
    function matchRule(rule, authorText, titleText) {
        if (rule.filter === 'all') {
            return rule.reg.test(authorText) || rule.reg.test(titleText);
        } else if (rule.filter === 'video') {
            return rule.reg.test(titleText);
        } else if (rule.filter === 'up') {
            return rule.reg.test(authorText);
        }
        return false;
    }

    // 构建控制面板
    function createPanel() {
        const panel = document.createElement('div');
        panel.style.cssText = `
            position: fixed; bottom: 80px; right: 20px; z-index: 99998;
            background: rgba(255, 255, 255, 0.85); backdrop-filter: blur(4px);
            padding: 12px; border-radius: 8px; box-shadow: 0 2px 12px rgba(0,0,0,0.15);
            border: 1px solid rgba(232, 232, 232, 0.7); width: 170px; font-family: sans-serif;
        `;
        panel.innerHTML = `
            <h4 style="margin: 0 0 8px 0; color: #333; font-size: 13px; text-align: center;">视频过滤器</h4>
            <input id="filter-input" type="text" placeholder="输入文本..." style="width: 100%; padding: 4px 6px; box-sizing: border-box; margin-bottom: 6px; border: 1px solid #ddd; border-radius: 4px; font-size: 12px;">
            <select id="filter-scope" style="width: 100%; padding: 4px 2px; box-sizing: border-box; margin-bottom: 6px; border: 1px solid #ddd; border-radius: 4px; font-size: 11px;">
                <option value="all">全部 (标题+UP主)</option>
                <option value="video">仅限视频标题</option>
                <option value="up">仅限UP主名称</option>
            </select>

            <div style="display: flex; gap: 4px; margin-bottom: 8px;">
                <button id="add-w-kw" style="flex: 1; padding: 4px 0; background: #52c41a; color: #fff; border: none; border-radius: 4px; cursor: pointer; font-size: 11px;">+ 白词</button>
                <button id="add-w-rg" style="flex: 1; padding: 4px 0; background: #237804; color: #fff; border: none; border-radius: 4px; cursor: pointer; font-size: 11px;">+ 白正则</button>
            </div>

            <div style="display: flex; gap: 4px; margin-bottom: 8px;">
                <button id="add-b-kw" style="flex: 1; padding: 4px 0; background: #ff4d4f; color: #fff; border: none; border-radius: 4px; cursor: pointer; font-size: 11px;">+ 黑词</button>
                <button id="add-b-rg" style="flex: 1; padding: 4px 0; background: #a8071a; color: #fff; border: none; border-radius: 4px; cursor: pointer; font-size: 11px;">+ 黑正则</button>
            </div>

            <div style="text-align: right; margin-bottom: 4px;">
                <button id="reset-rules-btn" style="padding: 2px 6px; background: #8c8c8c; color: #fff; border: none; border-radius: 3px; cursor: pointer; font-size: 10px;">重置默认值</button>
            </div>

            <div id="rules-list" style="max-height: 120px; overflow-y: auto; font-size: 11px; color: #666; border-top: 1px solid #eee; padding-top: 6px;"></div>
        `;
        document.body.appendChild(panel);

        const input = document.getElementById('filter-input');
        const scopeSelect = document.getElementById('filter-scope');
        const listContainer = document.getElementById('rules-list');

        const scopeLabels = { all: 'all', video: 'video', up: 'up' };

        // 渲染规则界面列表
        function renderList() {
            let html = '';
            if(savedRules.white.length > 0) {
                html += `<div style="color:#52c41a;font-weight:bold;margin-bottom:2px;">[白名单]</div>`;
                html += savedRules.white.map((r, idx) => `
                    <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 3px; padding-left: 4px;">
                        <span style="overflow: hidden; text-overflow: ellipsis; white-space: nowrap; max-width: 125px;" title="[${scopeLabels[r.filter || 'all']}] ${r.value}">[${scopeLabels[r.filter || 'all']}] · ${r.value}</span>
                        <span class="del-rule-btn" data-list="white" data-idx="${idx}" style="color: #ff4d4f; cursor: pointer; font-weight: bold; padding: 0 2px;">×</span>
                    </div>
                `).join('');
            }
            if(savedRules.black.length > 0) {
                html += `<div style="color:#ff4d4f;font-weight:bold;margin-top:4px;margin-bottom:2px;">[黑名单]</div>`;
                html += savedRules.black.map((r, idx) => `
                    <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 3px; padding-left: 4px;">
                        <span style="overflow: hidden; text-overflow: ellipsis; white-space: nowrap; max-width: 125px;" title="[${scopeLabels[r.filter || 'all']}] ${r.value}">[${scopeLabels[r.filter || 'all']}] · ${r.value}</span>
                        <span class="del-rule-btn" data-list="black" data-idx="${idx}" style="color: #ff4d4f; cursor: pointer; font-weight: bold; padding: 0 2px;">×</span>
                    </div>
                `).join('');
            }
            listContainer.innerHTML = html || '<div style="color:#999;text-align:center;">暂无规则</div>';
        }

        // 本地持久化并刷新过滤
        function saveAndReload() {
            GM_setValue('filter_rules', savedRules);
            compileRules();
            renderList();
            document.querySelectorAll('.bili-feed-card').forEach(c => c.classList.remove('filtered-processed'));
            filterBilibiliVideos();
        }

        // 添加规则事件绑定
        document.getElementById('add-w-kw').onclick = () => {
            if(!input.value.trim()) return;
            savedRules.white.push({ type: 'keyword', filter: scopeSelect.value, value: input.value.trim() });
            input.value = ''; saveAndReload();
        };
        document.getElementById('add-w-rg').onclick = () => {
            if(!input.value.trim()) return;
            savedRules.white.push({ type: 'regexp', filter: scopeSelect.value, value: input.value.trim() });
            input.value = ''; saveAndReload();
        };
        document.getElementById('add-b-kw').onclick = () => {
            if(!input.value.trim()) return;
            savedRules.black.push({ type: 'keyword', filter: scopeSelect.value, value: input.value.trim() });
            input.value = ''; saveAndReload();
        };
        document.getElementById('add-b-rg').onclick = () => {
            if(!input.value.trim()) return;
            savedRules.black.push({ type: 'regexp', filter: scopeSelect.value, value: input.value.trim() });
            input.value = ''; saveAndReload();
        };

        // 一键重置事件
        document.getElementById('reset-rules-btn').onclick = () => {
            if (confirm('确定要恢复到默认的黑白名单规则吗？')) {
                savedRules = JSON.parse(JSON.stringify(defaultRules));
                saveAndReload();
                showToast('已成功恢复默认规则');
            }
        };

        // 删除规则事件
        listContainer.onclick = (e) => {
            if (e.target.classList.contains('del-rule-btn')) {
                const listName = e.target.getAttribute('data-list');
                const idx = parseInt(e.target.getAttribute('data-idx'));
                savedRules[listName].splice(idx, 1);
                saveAndReload();
            }
        };

        renderList();
    }

    // 执行过滤逻辑
    function filterBilibiliVideos() {
        // 直接匹配最外层的包裹卡片容器
        const videoCards = document.querySelectorAll('.bili-feed-card:not(.filtered-processed)');
        let hiddenCountThisTime = 0;
        let lastHiddenName = '';

        videoCards.forEach(card => {
            card.classList.add('filtered-processed');

            const authorElement = card.querySelector('.bili-video-card__info--author');
            const titleElement = card.querySelector('.bili-video-card__info--tit');

            if (authorElement || titleElement) {
                const authorText = authorElement ? (authorElement.getAttribute('title') || authorElement.textContent.trim()) : '';
                const titleText = titleElement ? (titleElement.getAttribute('title') || titleElement.textContent.trim()) : '';

                // 白名单拦截
                const isWhite = whiteCompiled.some(rule => matchRule(rule, authorText, titleText));
                if (isWhite) {
                    return;
                }

                // 黑名单拦截
                const isBlack = blackCompiled.some(rule => matchRule(rule, authorText, titleText));
                if (isBlack) {
                    card.style.display = 'none';
                    hiddenCountThisTime++;
                    lastHiddenName = titleText || authorText;
                }
            }
        });

        if (hiddenCountThisTime > 0) {
            if (hiddenCountThisTime === 1) {
                showToast(`已隐藏视频: "${lastHiddenName}"`);
            } else {
                showToast(`快跑！已自动隐藏 ${hiddenCountThisTime} 个相关视频`);
            }
        }
    }

    // 初始化运行
    createPanel();
    filterBilibiliVideos();

    // 监听 DOM 变化节流
    let debounceTimer = null;
    const observer = new MutationObserver(() => {
        clearTimeout(debounceTimer);
        debounceTimer = setTimeout(() => {
            filterBilibiliVideos();
        }, 100);
    });

    observer.observe(document.body, {
        childList: true,
        subtree: true
    });
})();
```
