# Yt// ==UserScript==
// @name         YouTube Traditional Chinese Subtitle Fix
// @name:zh-TW   YouTube 繁體中文字幕修正
// @version      1.0
// @description  Fix YouTube's traditional Chinese subtitles by converting from simplified Chinese
// @match        https://www.youtube.com/*
// @grant        none
// @require      https://cdn.jsdelivr.net/npm/opencc-js@1.0.5/dist/umd/full.js
// @author       Frank0945 (original code), leVirve/Claude (userscript conversion)
// @homepage     https://github.com/Frank0945/fix-yt-traditional-chinese-subtitle
// @license      MIT
// ==/UserScript==

/*
 * This userscript is based on the work by Frank0945:
 * https://github.com/Frank0945/fix-yt-traditional-chinese-subtitle
 *
 * Original code is MIT licensed.
 * Conversion to userscript format was done by Claude, an AI assistant.
 */

(function() {
    'use strict';

    const FIXED_TAG_TRADITIONAL = `<span title="Fixed by 擴充「YouTube 繁體自動翻譯修正」" style="
      background: rgb(125 125 125 / 50%);
      margin-right: 3px;
      border-radius: 4px;
      padding: 2px 5px;
    ">修復</span>中文（繁體）`;
    const FIXED_TRADITIONAL = "修復中文（繁體）";
    const TRADITIONAL = "中文（繁體）";
    const TO = ">> ";

    const converter = OpenCC.Converter({ from: 'cn', to: 'twp' });

    const FILTER = {
        HANS: "tlang=zh-Hans",
        HANT: "tlang=zh-Hant",
    };
    const ORIGIN_TLANG = "origin_tlang=zh-Hant";

    // Intercept XMLHttpRequest
    const XHR = XMLHttpRequest.prototype;
    const open = XHR.open;
    const send = XHR.send;
    const setRequestHeader = XHR.setRequestHeader;

    XHR.open = function() {
        this._requestHeaders = {};
        if (arguments[1].includes(FILTER.HANT)) {
            arguments[1] = arguments[1].replace(FILTER.HANT, FILTER.HANS) + "&" + ORIGIN_TLANG;
        }
        return open.apply(this, arguments);
    };

    XHR.setRequestHeader = function(header, value) {
        this._requestHeaders[header] = value;
        return setRequestHeader.apply(this, arguments);
    };

    XHR.send = function() {
        this.addEventListener("load", function() {
            const url = this.responseURL;
            if (url.includes(FILTER.HANS) && url.includes(ORIGIN_TLANG)) {
                const converted = converter(this.responseText);
                Object.defineProperty(this, "responseText", {
                    writable: true,
                    value: converted,
                });
                console.log("[fix-yt-traditional-chinese-subtitle]: Converted successfully");
            }
        });
        return send.apply(this, arguments);
    };

    // Modify website button labels
    const addObserver = () => {
        const menu = document.querySelector("#movie_player .ytp-settings-menu");

        if (!menu) {
            // Check again after 100ms until the menu is in the DOM.
            setTimeout(addObserver, 100);
            return;
        }

        const observer = new MutationObserver(() => changeText(menu));

        observer.observe(menu, {
            childList: true,
            attributeFilter: ["style"],
        });
    };

    const changeText = (menu) => {
        const title = menu.querySelector(".ytp-panel-title");
        const items = menu.querySelectorAll(".ytp-menuitem");

        const isAutoPrefix = title?.textContent !== "自動翻譯";

        for (const item of items) {
            const elements = [
                item.querySelector(".ytp-menuitem-content"),
                item.querySelector(".ytp-menuitem-label"),
            ];

            for (const element of elements) {
                if (!element?.textContent) continue;

                if (isModified(element.textContent)) return;

                if (isCanBeModified(isAutoPrefix, element.textContent)) {
                    element.innerHTML = addTag(element.textContent);
                    return;
                }
            }
        }
    };

    const isModified = (text) => {
        return text === FIXED_TRADITIONAL || text.includes(TO + FIXED_TRADITIONAL);
    };

    const isCanBeModified = (isAutoPrefix, text) => {
        return (text === TRADITIONAL && !isAutoPrefix) || text.includes(TO + TRADITIONAL);
    };

    const addTag = (text) => {
        return text === TRADITIONAL
            ? FIXED_TAG_TRADITIONAL
            : text.replace(TO + TRADITIONAL, TO + FIXED_TAG_TRADITIONAL);
    };

    // Start observing
    window.addEventListener('load', addObserver);

    console.log("[fix-yt-traditional-chinese-subtitle]: Loaded successfully");
})();