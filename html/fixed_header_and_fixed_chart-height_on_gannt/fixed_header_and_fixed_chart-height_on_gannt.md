# ガントチャートの日付ヘッダーとチャート高さを固定する
## 説明
ガントチャートの表示項目が多い場合，チャートエリアが縦に長くなるため，下方にある項目の進捗を確認する際にはウィンドウを下方にスクロールすることになりますが，その際に日付ヘッダーが画面外に隠れてしまうため日付を確認することが困難になります．また，横スクロールバーも画面下部に位置しているため，横スクロールをする際には画面下部を表示する必要があり面倒です．

この問題に対処するために，JavaScript を使用することで，チャートエリアの高さを制限し，画面内に横スクロールバーを表示すると共に，スクロールした際に日付ヘッダーを画面上部に固定させる様にしました．  

尚，チャートエリアの高さ固定と日付ヘッダーの固定はチェックボックスで切り替えが可能です．  

注意点として，デザインが多少崩れます．また，IEでは横スクロール時やウィンドウサイズ変更時に動作が重たく感じられる可能性があります．  

## イメージ
### Before
例1: 項目が多いと横スクロールバーが画面外に隠れてしまう．  
![before](before_1.png)

例2: 横スクロールバーを表示するために画面下方にスクロールすると，日付ヘッダーが画面外に隠れてしまう．  
![before](before_2.png)

### After
チャート高さを制限し，日付ヘッダーを固定することで，横スクロールバーと日付ヘッダーを同時に表示出来るようになる．  
![after](after.png)

## 動作確認
- Redmine
  - 4.1.1
- ブラウザ
  - IE11
  - Chrome

## 設定
- パスのパターン: /gantt$
- 種別: HTML

## コード
```HTML
<script>
    //<![CDATA[
    $(function () {
        // 基本設定
        // settings
        const MIN_GANTT_HEIGHT = 300;
        const SCROLL_DELTA = 10;
        const LABEL_FIX_GANTT_HEIGHT = 'チャート高さ固定';
        const LABEL_FIX_HEADER = 'ヘッダー固定';

        // Redmine はデフォルトでスクロールバーが表示される仕様なので，
        // ウィンドウの幅からスクロールバーの幅を取得する
        const SCROLLBAR_WIDTH = window.innerWidth - document.documentElement.clientWidth;
        const INITIAL_HEIGHT = $('#gantt_area').height();
        const MOUSE_WHEEL_EVENT = 'onwheel' in document ? 'wheel'
            : 'onmousewheel' in document ? 'mousewheel' : 'DOMMouseScroll';

        let scroll_old = {top: 0, left: 0};
        let gantt_subjects_container = null;
        let gantt_subjects_columns_div = null;
        let gantt_subjects_columns = null;
        let gantt_hdr_selected_column_name = null;
        let gantt_hdr_first = null;


        var preprocessing = function () {
            gantt_subjects_container = $('div.gantt_subjects_container:first');
            gantt_subjects_columns_div = $(
                'td.gantt_subjects_column>div:first-child, ' +
                'td.gantt_selected_column>div:first-child');
            gantt_subjects_columns = $(
                'td.gantt_subjects_column, td.gantt_selected_column');
            gantt_hdr_selected_column_name = $('.gantt_hdr_selected_column_name');
            gantt_hdr_first = $('#gantt_area>.gantt_hdr:first');
            
            // 表示切替用のチェックボックスを追加
            var add_checkbox = function () {
                if ($('#fix_header').length === 0) {
                    $('#query_form_with_buttons>p.contextual').prepend(
                        '<input type="checkbox" id="fix_header" checked>' + 
                        '<label>' + LABEL_FIX_HEADER + '</label>')
                }
                if ($('#fix_gantt_height').length === 0) {
                    $('#query_form_with_buttons>p.contextual').prepend(
                        '<input type="checkbox" id="fix_gantt_height" checked>' + 
                        '<label>' + LABEL_FIX_GANTT_HEIGHT + '</label>'
                    )
                }
            }

            // チャートエリアの固定ヘッダー要素を追加
            var add_fixed_header = function () {
                $('#gantt_area').prepend(
                    '<div id="gantt_area_header_fixed_outer">' +
                    '<div id="gantt_area_header_fixed"></div></div>');
                $('#gantt_area_header_fixed').append(
                    $('#gantt_area .gantt_hdr').clone(true));
                $('#gantt_area_header_fixed_outer').css({
                    top: $('#gantt_area').offset().top,
                    zIndex: 50,
                    height: '74px',
                    overflow: 'hidden',
                    position: 'fixed',
                    width: $('#gantt_area').css('width'),
                });

                // 土日欄が下まで延長されてしまう現象を防ぐ
                $('#gantt_area_header_fixed .gantt_hdr').filter(
                    function () {return parseInt($(this).height()) > 72;}
                    ).css('height', '17px');
            }

            // チケット名欄の固定ヘッダー要素を追加
            var add_fixed_subjects_container = function () {
                $('#content div.gantt_subjects_container').prepend(
                    '<div id="gantt_subjects_container_fixed"></div>');
                $('#gantt_subjects_container_fixed').append(
                    $('#content div.gantt_subjects_container .gantt_hdr:first'));
                $('#gantt_subjects_container_fixed').css({
                    zIndex: 55,
                });
            }

            add_checkbox();
            if ($('#gantt_area_header_fixed').length === 0) {
                add_fixed_header();
            }
            if ($('#gantt_subjects_container_fixed').length === 0) {
                add_fixed_subjects_container();
            }
        }

        // --- define functions ---
        // チャートエリアの高さを設定
        var set_gantt_area_height = function () {

            var set_height = function (height) {
                $('#gantt_area').height(height);
                gantt_subjects_container.height(height - SCROLLBAR_WIDTH);
                gantt_subjects_columns_div.height(height - SCROLLBAR_WIDTH);
            }

            var reset_height = function () {
                set_height(INITIAL_HEIGHT);
                $('#content').css('min-height', '');
            }

            var window_height = $(window).height();

            // チェックボックスがオフの場合はチャート高さをリセットする
            if (!$('#fix_gantt_height').is(':checked')) {
                reset_height();
                return;
            } else {
                $('#content').css('min-height', 0);
            }

            // チャート高さを画面高さに合うように調整する
            var $next_main = $('#main').next();
            if ($next_main.css('display') !== 'none' && $next_main !== $('#footer')) {  // suppoort 2-pane mode
                var overflow_height = $('#content').parent().height() - $('#main').height();
            } else {
                var overflow_height = $(document).height() - window_height;
            }

            // チャートエリアの拡張が必要な場合，拡張量を算出する
            if (overflow_height <= 0) {
                var bottom = $('#content>.other-formats').offset().top
                     + $('#content>.other-formats').outerHeight(true);

                if ($next_main.css('display') !== 'none' && $next_main !== $('#footer')) {  // suppoort 2-pane mode
                    bottom += $next_main.outerHeight();
                }
                
                var overflow_height = bottom - $('#footer').offset().top;
                if (overflow_height >= 0) return;
            }

            // margin 1px (安定のため)
            overflow_height += 1;

            var current_height = $('#gantt_area').height();
            var max_height = current_height - overflow_height;

            // チャートエリアの高さが小さすぎる場合はチャート高さを制限する
            if (max_height < MIN_GANTT_HEIGHT) {
                set_height(MIN_GANTT_HEIGHT);
            } else {
                set_height(max_height);
            }
        }

        // 固定ヘッダー位置の設定
        var set_fixed_header_position = function(flg_left_only) {
            if (!$('#fix_header').is(':checked')) return
            if (typeof flg_left_only === 'undefined') flg_left_only = false

            $('#gantt_area_header_fixed').offset({left: gantt_hdr_first.offset().left});

            if (!flg_left_only) {
                var header_top = $('#gantt_area')[0].getBoundingClientRect().top;

                if (header_top < $('#main')[0].getBoundingClientRect().top) {
                    header_top = $('#main')[0].getBoundingClientRect().top;
                }

                if (header_top < 0 && $(document).scrollTop() > 0) {
                    header_top = 0;
                }

                $('#gantt_subjects_container_fixed').css({top: header_top});
                gantt_hdr_selected_column_name.parent().css({top: header_top});
                $('#gantt_area_header_fixed_outer').css({top: header_top});
            }
        }

        // ヘッダー固定状態とフロー状態を切り替える
        var set_header_state = function () {
            if (!$('#fix_header').is(':checked')) {
                $('#gantt_subjects_container_fixed').css({
                    position: 'absolute',
                    top: 0
                });
                gantt_hdr_selected_column_name.parent().css({
                    position: 'absolute',
                    top: 0
                });
                $('#gantt_area_header_fixed_outer').hide();
            } else {
                $('#gantt_subjects_container_fixed').css({
                    position: 'fixed',
                    width: $('#gantt_subjects_container_fixed').parent().css('width')  // support responsive design
                });
                gantt_hdr_selected_column_name.parent().css({
                    position: 'fixed',
                });

                var header_width = $('#gantt_area').width();
                if ($('#fix_gantt_height').is(':checked')) {
                    header_width -= SCROLLBAR_WIDTH;
                }
                $('#gantt_area_header_fixed_outer').css({
                    width: header_width
                });

                set_fixed_header_position();
                $('#gantt_area_header_fixed').css({left: 0});  // ?
                $('#gantt_area_header_fixed_outer').show();
            }
        }

        // チャート高さ固定状態と初期状態を切り替える
        var set_gantt_height_state = function () {
            if ($('#fix_gantt_height').is(':checked')) {
                $(
                    'div.gantt_subjects_container, ' + 
                    'div.gantt_selected_column_container').css({
                    borderBottom: '1px solid #c0c0c0'
                });
            } else {
                set_header_state();
                $(
                    'div.gantt_subjects_container, ' + 
                    'div.gantt_selected_column_container').css({
                    borderBottom: ''
                });
            }
        }

        // ガントチャートエリアのスクロールを同期
        var sync_gantt_scroll = function (top, left) {
            if (typeof top === 'undefined') top = $('#gantt_area').scrollTop();
            if (typeof left === 'undefined') left = $('#gantt_area').scrollLeft();

            gantt_subjects_container.scrollTop(top);
            gantt_subjects_columns_div.scrollTop(top);
            set_fixed_header_position(flg_left_only=true);

            // backup scroll position
            scroll_old.top = top;
            scroll_old.left = left;
        }

        // イベント追加用メソッド
        // （イベント重複防止のためラベルを付与）
        var add_event_with_label = function ($obj, event_name, event) {
            var label = 'fixed_gantt_plugin_event_added_' + event_name;
            if ($obj.data(label)) return;
            $obj.on(event_name, event)
            $obj.data(label, 1);
        }

        // イベント
        var add_events = function () {
            // ヘッダー固定チェックボックス用クリックイベント
            add_event_with_label($('#fix_header'), 'click',
                function () {
                    set_header_state();
                });

            // チャート高さ固定チェックボックス用クリックイベント
            add_event_with_label($('#fix_gantt_height'), 'click',
                function () {
                    set_gantt_area_height();
                    set_gantt_height_state();
                });

            // ウィンドウスクロールイベント
            add_event_with_label($(document), 'scroll', function () {
                set_header_state();
            });

            // スクロールイベント for 2-pane mode
            add_event_with_label($('#main_wrapper1'), 'scroll', function () {
                set_header_state();
            });

            // ガントチャートエリアスクロールイベント
            add_event_with_label($('#gantt_area'), 'scroll', function () {
                sync_gantt_scroll();
            });

            // 各オプションの幅変更イベント
            add_event_with_label(gantt_subjects_columns, 'resize',
                function () {
                    $(
                        '#gantt_subjects_container_fixed .gantt_hdr'
                    ).css('width', $('.gantt_subjects_column').css('width'));
                    set_header_state();
                });

            // フィルター，オプション関連ボタンクリックイベント
            var add_click_event_for_fiters_and_options = function () {
                var target_list = [
                    '#filters>legend',
                    '#options>legend',
                    '#draw_selected_columns'
                ]

                for (var i=0; i<target_list.length; i++) {
                    add_event_with_label(
                        $(target_list[i]), 'click',
                        function () {
                            window.setTimeout(function () {
                                set_gantt_area_height();
                                set_fixed_header_position();
                            }, 0);
                        });
                }
            }
            add_click_event_for_fiters_and_options();

            // ウィンドウリサイズイベント
            add_event_with_label(
                $(window), 'resize',
                function () {
                    set_gantt_area_height();
                    set_header_state();
                });

            // マウスホイールイベント
            // （チャートエリア外でのホイールによるスクロールを可能にする）
            var add_wheel_event = function () {
                var target_list = [
                    'td.gantt_subjects_column',
                    'td.gantt_project_column',
                    'td.gantt_selected_column'
                ]

                for (var i=0; i<target_list.length; i++) {
                    add_event_with_label($(target_list[i]), MOUSE_WHEEL_EVENT, function(event) {
                        if (event.originalEvent.wheelDelta > 0 || event.originalEvent.deltaY < 0 || event.originalEvent.detail < 0) {
                            // scroll up
                            $('#gantt_area').scrollTop(
                                $('#gantt_area').scrollTop() - SCROLL_DELTA);
                        }
                        else {
                            // scroll down
                            $('#gantt_area').scrollTop(
                                $('#gantt_area').scrollTop() + SCROLL_DELTA);
                        }
                    });
                }
            }
            add_wheel_event();

            // hideSidebar plugin 向け設定
            add_event_with_label(
                $('#hideSidebarButton'), 'click', function () {
                    set_header_state();
                });
        }

        // 各種関数の初回適用
        var initialize = function () {
            preprocessing();
            set_gantt_height_state();
            set_gantt_area_height();
            set_header_state();
            add_events();
        }

        var set_mutation_observer = function () {
            var timerId = null;
            var observer_table_child_list_change =
                new MutationObserver(function (mutations) {
                    // ignore svg
                    var flg = false;
                    for(var i=0; i<mutations.length; i++) {
                        if (mutations[i].target.tagName !== 'svg')  {
                            flg = true;
                            break;
                        }
                    }
                    if (!flg) return;

                    if (typeof timerId === 'number') {
                        clearTimeout(timerId);
                    }
                    timerId = setTimeout(function () {
                        $.when()
                            .then(initialize)
                            .then(function () {
                                // restore scroll position
                                $('#gantt_area').scrollTop(scroll_old.top);
                                $('#gantt_area').scrollLeft(scroll_old.left);
                                sync_gantt_scroll(
                                    scroll_old.top, scroll_old.left
                                );
                            });
                    }, 1000);
                });
            observer_table_child_list_change.observe(
            $('table.gantt-table:first')[0], {
                childList : true,
                subtree: true
            });

            var observer_rsize_main =
                new MutationObserver(function (mutations) {
                    set_gantt_area_height();
                    set_gantt_height_state();
                    set_fixed_header_position();
                });
            observer_rsize_main.observe(
                $('#main')[0], {
                    attributes : true
                });

            var observer_rsize_filter =
                new MutationObserver(function (mutations) {
                    set_gantt_area_height();
                    set_gantt_height_state();
                    set_fixed_header_position();
                });
            observer_rsize_filter.observe(
                $('#filters')[0], {
                    childList : true,
                    subtree: true
                });

            var observer_display_flash_notice =
                new MutationObserver(function (mutations) {
                    set_gantt_area_height();
                    set_gantt_height_state();
                    set_fixed_header_position();
                });
            if ($('#flash_notice').length > 0) {
                observer_display_flash_notice.observe(
                    $('#flash_notice')[0], {
                        attributes : true
                    });
            }
        }

        // initialze
        $.when()
            .then(initialize)
            .then(set_mutation_observer);

    });
    //]]>
</script>

<style>
    td.gantt_subjects_column>div:first-child, td.gantt_selected_column>div:first-child{
        overflow: hidden;
    }
    table.gantt-table td:last{
        /* #gantt_area.parent() */
        vertical-align: top;
    }
    #gantt_area{
        border-left: 1px solid #c0c0c0;
        margin-left: 1px;
    }
    div.gantt_subjects_column:first {
        /* div.gantt_subjects_container:first.parent() */
        vertical-align: top;
    }
    td.gantt_subjects_column, td.gantt_selected_column{
        vertical-align: top;
    }
    td.gantt_subjects_column>div:first-child, td.gantt_selected_column>div:first-child{
        z-index: 60;
    }
    td.gantt_selected_column div.gantt_hdr:nth-child(even) {
        /* .gantt_hdr_selected_column_name.parent() */
        z-index: 70;
    }
</style>
```
