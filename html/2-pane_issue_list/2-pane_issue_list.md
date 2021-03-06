# チケット一覧画面＆ガントチャート画面を2ペイン化
## 説明
対象の画面を2ペイン化し，対象の画面とチケット詳細を1画面内に同時に表示します．  
これにより，対象の画面を確認しながら，チケット詳細の確認と編集が可能になります．  
また，チケット詳細画面で行った変更は都度対象の画面に反映されます．  
現在，チケット一覧画面，ガントチャート画面に対応しています．

## 動作確認済環境
### Redmine
- 4.1.1
### ブラウザ
- Chrome 86.0.4240.75
- Firefox 81.0.2
- IE11
### プラグイン
- [redmine_issue_dynamic_edit](https://www.redmine.org/plugins/redmine_issue_dynamic_edit)
- [sidebar_hide](https://www.redmine.org/plugins/sidebar_hide)

## 注意
- HTMLのDOM階層を変更しているため，その他のプラグイン等が正常に動作しなくなる可能性があります．
- チケット詳細画面内のサイドバーにはウォッチャー以外の項目は表示されません．

## イメージ
![after](after.png)
![after2](after2.png)

## View Customize plugin 設定
- パスのパターン: /(issues|gantt)$
- 種別: HTML

## コード
```HTML
<script>
    //<![CDATA[
    $(function () {
        // --- config/設定 ---
        // 2ペイン表示モードを初期状態とする
        var SET_INITIAL_STATE_TO_2PANEMODE = false;
        // 表示モードをCookieに保持する（Cookieに保存された値が優先されます）
        var USE_COOKIE = true;
        // 2ペイン表示モード時のメインペインの高さ
        var INITIAL_HEIGHT_OF_MAIN_PANE = '30vh';
        // 2ペイン表示モード時のメインペインの最小高さ
        var MIN_HEIGHT_OF_MAIN_PANE = 50;
        // ----------------


        if (document.location.pathname.match(/issues$/g)) {
            var targetAnchor = function () {
                return $('#content table.list.issues').find('td.id a, td.subject a');
            }
        } else if (document.location.pathname.match(/gantt$/g)) {
            var targetAnchor = function () {
                return $('table.gantt-table').find(
                    'div.gantt_subjects a.issue, #gantt_area span.tip a.issue');
            }
        } else {
            return;
        }


        // ----- define functions -----
        // https://www.w3schools.com/js/js_cookies.asp
        function setCookie(cname, cvalue, exdays) {
            var d = new Date();
            d.setTime(d.getTime() + (exdays * 24 * 60 * 60 * 1000));
            var expires = 'expires=' + d.toUTCString();
            document.cookie = cname + '=' + cvalue + ';' + expires + ';path=/';
        }


        // https://www.w3schools.com/js/js_cookies.asp
        function getCookie(cname) {
            var name = cname + '=';
            var decodedCookie = decodeURIComponent(document.cookie);
            var ca = decodedCookie.split(';');
            for (var i = 0; i < ca.length; i++) {
                var c = ca[i];
                while (c.charAt(0) == ' ') {
                    c = c.substring(1);
                }
                if (c.indexOf(name) == 0) {
                    return c.substring(name.length, c.length);
                }
            }
            return '';
        }


        var showDetail = function () {
            var height = $('#main').data('height');
            if (typeof height === 'undefined' || height === '') {
                height = INITIAL_HEIGHT_OF_MAIN_PANE;
            }
            $('#wrapper3').css('height', '100vh'); // support IE11
            $('#main').css('max-height', height);
            $('#main_wrapper1').css('max-height', height);
            $('#main_wrapper1').css('overflow-y', 'scroll');
            $('#main_wrapper1').css('width', ''); // support IE11
            setTimeout(function () {
                $('#main_wrapper1').css('width', '100%');
            }); // support IE11
            $('#iframe_issue_detail_wrapper').show();
            $('#main>div.ui-resizable-handle').show();
        }


        var hideDetail = function () {
            $('#iframe_issue_detail_wrapper').hide();
            $('#main>div.ui-resizable-handle').hide();
            $('#wrapper3').css('height', ''); // support IE11
            $('#main').css('max-height', '');
            $('#main').css('height', ''); // for resizeable
            $('#main_wrapper1').css('max-height', '');
            $('#main_wrapper1').css('overflow-y', '');
        }


        var visibleIframe = function () {
            $('#iframe_issue_detail').css('visibility', 'visible');
            $('#ajax-indicator').hide();
        }


        var hideIframe = function () {
            $('#iframe_issue_detail').css('visibility', 'hidden');
            $('#ajax-indicator').show();
        }


        var enable2PaneMode = function () {
            if ($('#iframe_issue_detail').attr('src') === '') {
                hideDetail();
            } else {
                showDetail();
            }

            targetAnchor().each(function () {
                // disable the original link
                if ($(this).attr('href') !== 'javascript:void(0)') {
                    $(this).data('href', $(this).attr('href'));
                    $(this).attr('href', 'javascript:void(0)');
                }

                $(this).on('click', updateIframe);
            });

            if (USE_COOKIE) setCookie('2pane_mode', 'true', 100);
        }


        var disable2PaneMode = function () {
            hideDetail();

            targetAnchor().each(function () {
                // restore the original link
                if (typeof $(this).data('href') !== 'undefined') {
                    $(this).attr('href', $(this).data('href'));
                }

                $(this).off('click', updateIframe);
            });

            if (USE_COOKIE) setCookie('2pane_mode', 'false', 100);
        }


        var updateState = function () {
            if ($('#cb_2pane_mode').prop('checked')) {
                enable2PaneMode();
            } else {
                disable2PaneMode();
            }
        }


        var updateIframe = function () {
            if (!$('#cb_2pane_mode').prop('checked')) {
                updateState();
                return
            }

            var $iframe = $('#iframe_issue_detail');

            if ($iframe.attr('src') === $(this).data('href')) {
                if ($('#iframe_issue_detail_wrapper').css('display') !== 'none') {
                    hideDetail();
                } else {
                    showDetail();
                }
            } else {
                hideIframe();
                $iframe.data('count', 0);
                $iframe.attr('src', $(this).data('href'));
            }
        }


        var updateIssueList = function () {
            var url = document.location.href;
            $.get(url).done(function (data) {
                var contentNew = $('table.list.issues', $(data)).first().parent().html();
                $('table.list.issues:first').replaceWith($(contentNew));
                updateState();
            });
        }


        var updateGanttTable = function () {
            var url = document.location.href;
            var style_display = $('.gantt_selected_column').css('display');
            var $gantt_table_org = $('#content table.gantt-table:first');

            // insert temporary gantt-table
            var $gantt_table_temp = $('<table class="gantt-table"></table>');
            $gantt_table_temp.insertAfter($gantt_table_org);
            $gantt_table_temp.hide();

            var update_gantt_object = function (selector, keep_style, replace) {
                if(typeof keep_style === 'undefined') keep_style = [];  // support ie11
                if(typeof replace === 'undefined') replace = true;  // support ie11

                var list_$org = $gantt_table_org.find(selector);
                var list_$temp = $gantt_table_temp.find(selector);

                for (i = 0; i<list_$org.length; i++) {
                    var $org = $(list_$org[i]);
                    var $temp = $(list_$temp[i]);
                    for (j=0; j<keep_style.length; j++) {
                        $temp.css(keep_style[j], $org.css(keep_style[j]));
                    }
                    if (replace) {
                        $($org).replaceWith($temp);
                    }
                }
            }

            var afterLoad = function () {

                /*
                target for update
                1. .gantt_subjects_container
                2. .gantt_selected_column_container
                3. #gantt_area
                4. restoring events
                */

                // 1. update gantt subjects container
                update_gantt_object(
                    '.gantt_subjects_container .gantt_hdr',
                    ['width'], false)
                update_gantt_object('.gantt_subjects_container', ['width']);
                // set width (copy from gantt.js)
                $('.issue-subject, .project-name, .version-name').each(function () {
                    $(this).width($(".gantt_subjects_column").width()-$(this).position().left);
                });

                // 2. update selected columns container
                update_gantt_object(
                    '.gantt_selected_column_container .gantt_hdr',
                    ['width'], false)
                update_gantt_object(
                    '.gantt_selected_column_container', ['width']);

                // 3. update gantt area
                update_gantt_object('#gantt_area');

                // 4. restor the events (copy from gantt.js)
                draw_gantt = null;
                drawGanttHandler()
                $('div.gantt_subjects .expander').on('click', ganttEntryClick);

                // finally
                $gantt_table_temp.remove();
                updateState();
            }

            $gantt_table_temp.load(
                url + ' table.gantt-table > tbody', afterLoad
            );
        }


        var updateMainTable = function () {
            if (document.location.pathname.match(/issues$/g)) {
                updateIssueList();
            } else if (document.location.pathname.match(/gantt$/g)) {
                updateGanttTable();
            }
        }


        var getIframeWindow = function () {
            var $iframe = $('#iframe_issue_detail');
            if (typeof $iframe === 'undefined') return;
            var window_iframe = $iframe[0].contentWindow;
            if (window_iframe === window) return;
            return window_iframe;
        }


        var setSidebarWidth = function () {
            var $iframe = $('#iframe_issue_detail');
            var window_iframe = getIframeWindow();
            if (typeof window_iframe === 'undefined') return;
            $('#sidebar', $iframe.contents()).css('width', $('#sidebar').css('width'));
        }
        $(window).on('resize', function () {
            setSidebarWidth();
        });


        // ----- object settings -----
        // add checkbox
        if (USE_COOKIE) {
            SET_INITIAL_STATE_TO_2PANEMODE = getCookie('2pane_mode') === 'true';
        }
        var initialStateCheckedStr = SET_INITIAL_STATE_TO_2PANEMODE ? 'checked' : '';
        var $checkBoxBox = $('<span><input ' + initialStateCheckedStr +
            ' id="cb_2pane_mode" type="checkbox"><label>2-Pane Mode</label></span>');
        $('#content div.contextual:first').prepend($checkBoxBox);
        $checkBox = $('#cb_2pane_mode');
        $checkBoxBox.children('label').on('click', function () {
            $checkBox.prop('checked', !$checkBox.prop('checked'));
            $checkBox.change();
        });
        $checkBox.on('change', function () {
            updateState();
        });


        // move sidebar & content to main wrapper
        var main_wrapper1 = $('<div id="main_wrapper1"></div>');
        var main_wrapper2 = $('<div id="main_wrapper2"></div>');
        main_wrapper2.prependTo(main_wrapper1);
        main_wrapper1.prependTo('#main');
        $('#sidebar').appendTo(main_wrapper2);
        $('#content').appendTo(main_wrapper2);
        $('main').css('flex-direction', 'column');


        // create iframe
        var $iframe = $('<iframe id="iframe_issue_detail", src=""></iframe>');
        $iframe.css('width', '100%');
        $iframe.on('load', function () {
            if ($iframe.attr('src') === '') {
                $('#ajax-indicator').hide();
                return;
            }

            hideIframe();

            // check location
            var pathname = $iframe[0].contentWindow.location.pathname;
            if (pathname !== $iframe.attr('src')) {
                window.location.href = pathname;
                return;
            }

            // hide duplicate contents
            $('#top-menu', $iframe.contents()).hide();
            $('#header', $iframe.contents()).hide();
            $('#footer', $iframe.contents()).hide();

            // hide all but wathcers in the sidebar
            $('#sidebar>*:not(#watchers)', $iframe.contents()).hide();

            // observe update detail
            var updateBuffer
            var observerUpdateDetail =
                new MutationObserver(function (mutations) {
                    mutations.forEach(function (mutationRecord) {
                        if (typeof updateBuffer === 'number') {
                            clearTimeout(updateBuffer);
                        }
                        if (!$(mutationRecord.target).is(':visible')) {
                            updateBuffer = setTimeout(function () {
                                updateMainTable();
                            }, 1000);
                        }
                    });
                });
            observerUpdateDetail.observe(
                $('#ajax-indicator', $iframe.contents())[0], {
                    attributes: true,
                    attributeFilter: ['style']
                });

            if (typeof $iframe.data('count') === 'undefined') {
                $iframe.data('count', 1);
            } else {
                $iframe.data('count', $iframe.data('count') + 1);
            }
            if ($iframe.data('count') === 1) {
                $iframe[0].contentWindow.addEventListener('beforeunload', function (e) {
                    $iframe.css('visibility', 'hidden');
                    $('#ajax-indicator').show();
                });
                showDetail();
            } else {
                updateMainTable();
            }

            if ($('#sidebar').is(':visible') !== $('#sidebar', $iframe.contents()).is(':visible') &&
                typeof $('#hideSidebarButton') === 'undefined' ) {
                if ($('#sidebar').is(':visible')) {
                    $('#sidebar', $iframe.contents()).show();
                } else {
                    $('#sidebar').show();
                }
            }

            visibleIframe();
            setSidebarWidth();
        });


        // append iframe to main
        var $iframeWrapper = $('<div id="iframe_issue_detail_wrapper"></div>');
        $iframeWrapper.hide();
        $('#main').after($iframeWrapper.append($iframe));


        // ----- additional settings -----
        // resizeable
        $('#main').resizable({
            handles: 's',
            minHeight: MIN_HEIGHT_OF_MAIN_PANE,
            start: function () {
                $("#wrapper3").each(function (index, element) {
                    var d = $('<div class="iframe_cover"></div>');
                    d.css({
                        'z-index': 89,
                        'position': 'absolute',
                        'width': '100%',
                        'top': 0,
                        'left': 0,
                    });
                    d.height($(element).height());
                    $(element).append(d);
                });
            },
            stop: function () {
                $('.iframe_cover').remove();
            },
            resize: function (event, ui) {
                $('#main').css('max-height', ui.size.height);
                $('#main_wrapper1').css('max-height', ui.size.height);
                $('#main').data('height', ui.size.height);
            },
            create: function (event, ui) {
                $('#main>.ui-resizable-handle').on('dblclick', function () {
                    hideDetail();
                });
            }
        });


        // support hide_sidebar plugin
        // https://www.redmine.org/plugins/sidebar_hide
        if (typeof window.hideSideBar === 'function') {
            // replace function in parent window
            window.hideSideBarOrg = window.hideSideBar;
            window.hideSideBar = function () {
                window.hideSideBarOrg();
                if ($iframe.is(':visible')) {
                    $iframe[0].contentWindow.hideSideBarOrg();
                } else {
                    $iframe.attr('src', ''); // clear iframe
                }
            }

            // replace function in iframe
            $iframe.on('load', function () {
                if ($iframe.attr('src') === '') return;
                var window_iframe = getIframeWindow();
                if (typeof window_iframe === 'undefined') return;
                window_iframe.hideSideBarOrg = window_iframe.hideSideBar;
                window_iframe.hideSideBar = function () {
                    window_iframe.hideSideBarOrg();
                    window.parent.hideSideBarOrg();
                }
                if ($('#sidebar').is(':visible') !== $('#sidebar', $iframe.contents()).is(':visible')) {
                    window_iframe.hideSideBarOrg();
                }
            });
        }


        // ----- initialize -----
        updateState();
    });
    //]]>
</script>

<style>
    #main_wrapper1 {
        display: flex;
        flex-direction: column;
        flex-grow: 1;
        width: 100%;  /* support IE11 */
    }

    #main_wrapper2 {
        display: flex;
        flex-direction: row-reverse;
        flex-grow: 1;
        width: 100%;  /* support IE11 */
    }

    #iframe_issue_detail {
        flex-grow: 1;
        border-width: 0;
        min-height: 100%;  /* support IE11 */
        overflow-y: scroll;
    }

    #iframe_issue_detail_wrapper {
        flex-grow: 1;
        display: flex;
        z-index: 15;  /* support hide_sidebar plugin */
    }

    #main>div.ui-resizable-handle{
        background-color: #e4e4e4;
    }
</style>
```
