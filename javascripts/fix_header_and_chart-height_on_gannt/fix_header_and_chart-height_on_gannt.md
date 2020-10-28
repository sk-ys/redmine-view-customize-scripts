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
- 種別: JavaScript

## コード
```HTML
<script>
    //<![CDATA[
    $(function () {
        // 基本設定
        // settings
        var MIN_GANTT_HEIGHT = 750;
        var SCROLLBAR_WIDTH = 17;
        var SCROLL_DELTA = 100;

        var initial_height = $('#gantt_area').height();
        var scroll_top_old = 0;

        var gantt_subjects_container = null;
        var gantt_subjects_columns_div = null;
        var gantt_subjects_columns = null;
        var gantt_hdr_selected_column_name = null;
        var gantt_hdr_first = null;


        var preprocessing = function () {
            gantt_subjects_container = $('div.gantt_subjects_container:first');
            gantt_subjects_columns_div = $(
                'td.gantt_subjects_column>div:first-child, ' +
                'td.gantt_selected_column>div:first-child');
            gantt_subjects_columns = $(
                'td.gantt_subjects_column, td.gantt_selected_column');
            gantt_hdr_selected_column_name = $('.gantt_hdr_selected_column_name');
            gantt_hdr_first = $('#gantt_area>.gantt_hdr:first');

            // スタイルに関する設定
            // style settings
            /*
            gantt_subjects_container.css('overflow', 'hidden');
            gantt_subjects_columns_div.css('overflow', 'hidden');
            $('#gantt_area').parent().css({
                verticalAlign: 'top',
            });
            $('#gantt_area').css({
                borderLeft: '1px solid #c0c0c0',
                marginLeft: '1px'
            });
            gantt_subjects_container.parent().css('vertical-align', 'top');
            gantt_subjects_columns.css('vertical-align', 'top');
            
            gantt_subjects_columns_div.css('z-index', 60);
            gantt_hdr_selected_column_name.parent().css('z-index', 70);
            */
            
            // 表示切替用のチェックボックスを追加
            var add_checkbox = function () {
                if ($('#fix_header').length === 0) {
                    $('#query_form_with_buttons>p.contextual').prepend(
                        '<input type="checkbox" id="fix_header" checked>' + 
                        '<label>ヘッダー固定</label>')
                }
                if ($('#fix_gantt_height').length === 0) {
                    $('#query_form_with_buttons>p.contextual').prepend(
                        '<input type="checkbox" id="fix_gantt_height" checked>' + 
                        '<label>チャート高さ固定</label>'
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
                    paddingLeft: 1,
                    zIndex: 50,
                    height: '74px',
                    overflow: 'hidden',
                    position: 'fixed',
                    width: $('#gantt_area').css('width'),
                });

                    // 土日欄が下まで延長されてしまう現象を防ぐ
                $('#gantt_area_header_fixed .gantt_hdr').filter(
                    function () {return $(this).height() > 72;}).css('height', '17px');
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

        // define function
        // チャートエリアの高さを設定する関数を定義
        var set_gant_area_height = function (flg_2nd_try) {
            if (typeof flg_2nd_try === 'undefined') flg_2nd_try = false;
            var window_height = $(window).height();
            // var window_height = $('#top-menu').height() + $('#header').height() + $('#main').height();

            // チェックボックスがオフの場合はチャート高さを初期値とする
            if (!$('#fix_gantt_height').is(':checked')) {
                $('#gantt_area').height(initial_height);
                gantt_subjects_container.height(initial_height);
                gantt_subjects_columns_div.height(initial_height);
                return
            }

            // Windowの高さが小さすぎる場合はチャート高さをWindowの高さとする
            if (window_height < MIN_GANTT_HEIGHT) {
                $('#gantt_area').height(window_height);
                gantt_subjects_container.height(window_height);
                gantt_subjects_columns_div.height(window_height);
                return
            }

            // チャート高さを画面高さに合うように調整する
            // var overflow_height = $(document).height() - window_height;
            if ($('#main').next().position()['top'] > 0 && $('table.gantt-table').position()['top'] !== $('#main').next().position()['top']) {
                var overflow_height = $('table.gantt-table').position()['top'] - $('#main').next().position()['top'];
            } else {
                // var overflow_height = window_height - $('#footer').height() - $('table.gantt-table').position()['top'];
                // var overflow_height = window_height - $('#footer').height();
                // var overflow_height = $(document).height() - window_height;
                var overflow_height = $(document).height() - window_height;
            }

            var current_height = $('#gantt_area').height();
            
            if (overflow_height <= 0) {
                if (flg_2nd_try) return
                // $('#gantt_area').height(window_height);
                var max_height = window_height;
            } else {
                var max_height = current_height - overflow_height;
            }

            $('#gantt_area').height(max_height);
            gantt_subjects_container.height(max_height - SCROLLBAR_WIDTH);
            gantt_subjects_columns_div.height(max_height - SCROLLBAR_WIDTH);

            if (overflow_height <= 0) {
                set_gant_area_height(true);
            }
        }

        // ヘッダー固定状態とフロー状態を切り替える
        var set_state_header = function () {
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
                    top: $('#gantt_area').offset().top,
                    width: $('#gantt_subjects_container_fixed'
                        ).parent().css('width')  // support responsive design
                });
                gantt_hdr_selected_column_name.parent().css({
                    position: 'fixed',
                    top: $('#gantt_area').offset().top
                });
                $(
                    '#gantt_area_header_fixed_outer'
                ).width($('#gantt_area').width() - SCROLLBAR_WIDTH);
                $('#gantt_area_header_fixed_outer').show();
            }
        }

        // チャート高さ固定状態と初期状態を切り替える
        var set_state_gantt_height = function () {
            if (!$('#fix_gantt_height').is(':checked')) {
                $('#fix_header').prop('checked', false);
                set_state_header();
                $(
                    'div.gantt_subjects_container, ' + 
                    'div.gantt_selected_column_container').css({
                    borderBottom: ''
                });
            } else {
                $(
                    'div.gantt_subjects_container, ' + 
                    'div.gantt_selected_column_container').css({
                    borderBottom: '1px solid #c0c0c0'
                });
            }
        }

        // 固定ヘッダーの表示設定
        var set_position_fixed = function(flg_left_only){
            if (!$('#fix_header').is(':checked')) return
            if (typeof flg_left_only === 'undefined') flg_left_only = false
            $('#gantt_area_header_fixed').offset(
                {left: gantt_hdr_first.offset().left});
            if (!flg_left_only) {
                $(
                    '#gantt_subjects_container_fixed'
                ).offset({top: $('#gantt_area').offset().top});
                gantt_hdr_selected_column_name.parent().offset(
                    {top: $('#gantt_area').offset().top}
                );
                $(
                    '#gantt_area_header_fixed_outer'
                ).offset({top: $('#gantt_area').offset().top});
            }
        }


        // イベント追加用メソッド
        var add_event = function ($obj, event_name, event) {
            var tag_name = 'event_added_' + event_name;
            if ($obj.data(tag_name)) return;
            $obj.on(event_name, event)
            $obj.data(tag_name, 1);
        }


        var add_events = function () {
            // ヘッダー固定チェックボックス用クリックイベント
            add_event($('#fix_header'), 'click',
                function () {
                    if ($(this).is(':checked') && !$('#fix_gantt_height').is(':checked')) {
                        $('#fix_gantt_height').prop('checked', true);
                        set_state_gantt_height();
                        set_gant_area_height();
                    }
                    set_state_header();
                    set_position_fixed();
                });


            // チャート高さ固定チェックボックス用クリックイベント
            add_event($('#fix_gantt_height'), 'click',
                function () {
                    set_gant_area_height();
                    set_state_gantt_height();
                });

            // ウィンドウスクロールイベント
            add_event($(document), 'click', set_position_fixed);

            // ガントチャートエリアスクロールイベント
            add_event($('#gantt_area'), 'scroll',
                function () {
                    var scroll_top = $('#gantt_area').scrollTop();
                    if (scroll_top_old !== scroll_top) {
                        gantt_subjects_container.scrollTop(scroll_top);
                        gantt_subjects_columns_div.scrollTop(scroll_top);
                    } else {
                        set_position_fixed(flg_left_only=true);
                    }
                    scroll_top_old = scroll_top;
                });

            // 各オプションの幅変更イベント
            add_event(gantt_subjects_columns, 'resize',
                function () {
                    $(
                        '#gantt_subjects_container_fixed .gantt_hdr'
                    ).css('width', $('.gantt_subjects_column').css('width'));
                    set_state_header();
                    set_position_fixed(flg_left_only=true);
                });

            // フィルター，オプション関連ボタンクリックイベント
            var add_fiter_and_option_event = function () {
                var target_list = [
                    '#filters>legend',
                    '#options>legend',
                    '#draw_selected_columns'
                ]

                for (var i=0; i<target_list.length; i++) {
                    add_event(
                        $(target_list[i]), 'click',
                        function () {
                            window.setTimeout(function () {
                                set_gant_area_height();
                                set_position_fixed();
                            }, 10);
                        });
                }
            }
            add_fiter_and_option_event()

            // ウィンドウリサイズイベント
            add_event(
                $(window), 'resize',
                function () {
                    set_gant_area_height();
                    set_state_header();
                    set_position_fixed();
                });

            // マウスホイールイベント（チャートエリア外でのスクロールを可能にする）
            // wheel event
            var add_wheel_event = function () {
                var target_list = [
                    'td.gantt_subjects_column',
                    'td.gantt_project_column',
                    'td.gantt_selected_column'
                ]

                for (var i=0; i<target_list.length; i++) {
                    add_event($(target_list[i]), 'mousewheel DOMMouseScroll',
                        function(event){
                        if (event.originalEvent.wheelDelta > 0 || event.originalEvent.detail < 0) {
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
            add_event($('#hideSidebarButton'), 'click', set_state_header);
        }

        // 各種関数の初回適用
        var initialize = function () {
            console.log('initialize');
            window.setTimeout(function () {
                preprocessing();
                set_state_gantt_height();
                set_gant_area_height();
                set_state_header();
                set_position_fixed();
                add_events();
            }, 10);
        }


        var updateBuffer = null;
        var observer =
            new MutationObserver(function (mutations) {
                if (typeof updateBuffer === 'number') {
                    clearTimeout(updateBuffer);
                }
                updateBuffer = setTimeout(initialize, 1000);
            });
        observer.observe(
            $('table.gantt-table:first')[0], {
                childList : true,
                subtree: true
            });

        initialize();
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
    td.gantt_selected_column div.gantt_hdr:nth-child(even){
        /* .gantt_hdr_selected_column_name.parent() */
        z-index: 70;
    }
</style>
```
