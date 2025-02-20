<script setup lang="ts">
import { nextTick, onMounted, reactive, ref, watch } from 'vue';
import { Window, Ombra, Tools, FS } from '~/api'
import { useAppListStore } from '~/stores/appList';
import { useConfigStore } from '~/stores/config';
const props = defineProps(['main_input', 'cur_focus_app']);
const emit = defineEmits(['update:cur_focus_app']);

interface AppInfoExt extends AppInfo {
    is_match: boolean, //当前是否被匹配上了
    show_name: string, //要显示的名字
}

const configStore = useConfigStore();

const search_result_list = reactive([]) as Array<AppInfoExt>; //每次的搜索结果

const recommend_list = reactive([]) as Array<AppInfoExt>; //最多8个推荐应用

const search_count = ref(0); //搜索结果的数量

let search_result_is_expand = ref(false); //记录搜索结果是否为展开状态

const applistStore = useAppListStore();

onMounted(async () => {
    await init_data();
    search(); //初始化操作
});

//当app列表发送变化时，自动刷新app面板数据
let watch_applist_timer: NodeJS.Timeout | undefined;
watch(() => applistStore.applist.length, () => {
    if (watch_applist_timer) clearTimeout(watch_applist_timer);
    watch_applist_timer = setTimeout(async () => {
        await init_data();
        search();
    }, 200);
});

async function init_data() {
    let appinfo = await configStore.read_appinfo();
    const app_list = applistStore.applist;
    for (let i = 0; i < app_list.length; i++) {
        let f = true;
        for (let j = 0; j < appinfo.length; j++) {
            if (appinfo[j].name == app_list[i].name) {
                app_list[i].weight = appinfo[j].weight;
                f = false;
                break;
            }
        }
        if (f) {
            app_list[i].weight = 0;
            appinfo.push({
                name: app_list[i].name,
                weight: 0
            })
        }
    }
}

function click_app() {
    if (search_result_list.length == 0) {
        fun_open_app(recommend_list[props.cur_focus_app], false);
    } else {
        if (props.cur_focus_app >= search_result_list.length) {
            fun_open_app(recommend_list[props.cur_focus_app % search_result_list.length], false);
            return;
        }
        fun_open_app(search_result_list[props.cur_focus_app], true);
    }
}
async function move(dire: String) {
    let all_length = search_result_list.length + recommend_list.length;

    //没有任何可用应用，直接返回
    if (all_length == 0) return;

    switch (dire) {
        case 'up':
            {
                //如果两者结果都存在
                if (search_result_list.length && recommend_list.length) {
                    //如果当前在搜索结果内
                    if (props.cur_focus_app < search_result_list.length) {
                        //如果在最前面，则直接跳到最后
                        if (props.cur_focus_app == 0) {
                            emit('update:cur_focus_app', all_length - 1);
                        } else if (props.cur_focus_app < 8) { //如果在第一行，则每次前进一个
                            emit('update:cur_focus_app', props.cur_focus_app - 1);
                        } else { //否则每次前进一行
                            emit('update:cur_focus_app', props.cur_focus_app - 8);
                        }
                    } else {//如果当前在推荐结果内
                        //如果在推荐结果第一行
                        if (props.cur_focus_app < search_result_list.length + 8) {
                            let offset = props.cur_focus_app - search_result_list.length;
                            let pos = (Math.floor(search_result_list.length / 8) - 1) * 8 + offset;
                            emit('update:cur_focus_app', pos);
                        } else { //否则直接每次-8
                            emit('update:cur_focus_app', props.cur_focus_app - 8);
                        }
                    }
                } else {//如果搜索应用、推荐应用只有一个有结果
                    //如果在最前面，则直接跳到最后
                    if (props.cur_focus_app == 0) {
                        emit('update:cur_focus_app', all_length - 1);
                    } else if (props.cur_focus_app < 8) { //如果在第一行，则每次前进一个
                        emit('update:cur_focus_app', props.cur_focus_app - 1);
                    } else { //否则每次前进一行
                        emit('update:cur_focus_app', props.cur_focus_app - 8);
                    }
                }
            }
            break;
        case 'down':
            {
                //如果两者结果都存在
                if (search_result_list.length && recommend_list.length) {
                    //如果当前在搜索结果内
                    if (props.cur_focus_app < search_result_list.length) {
                        let pos = Math.floor((search_result_list.length - 1) / 8) * 8; //最后一行的起始数字(下标)
                        if (props.cur_focus_app >= pos) { //如果在最后一行，则根据偏移值调准到推荐结果
                            let offset = props.cur_focus_app - pos;
                            if (search_result_list.length + offset >= all_length) {
                                emit('update:cur_focus_app', all_length - 1);
                            } else {
                                emit('update:cur_focus_app', search_result_list.length + offset);
                            }
                        } else { //不在最后一行，每次直接-8
                            emit('update:cur_focus_app', all_length + 8);
                        }
                    } else {//如果当前在推荐结果内
                        //如果在推荐结果最后一行
                        if (props.cur_focus_app >= all_length - 8) {
                            //如果在最后一个，则直接跳到第一个
                            if (props.cur_focus_app == all_length - 1) {
                                emit('update:cur_focus_app', 0);
                            } else { //否则每次+1
                                emit('update:cur_focus_app', props.cur_focus_app + 1);
                            }
                        } else { //否则直接每次+8
                            emit('update:cur_focus_app', props.cur_focus_app + 8);
                        }
                    }
                } else {//如果搜索应用、推荐应用只有一个有结果
                    //如果在推荐结果最后一行
                    if (props.cur_focus_app >= all_length - 8) {
                        //如果在最后一个，则直接跳到第一个
                        if (props.cur_focus_app == all_length - 1) {
                            emit('update:cur_focus_app', 0);
                        } else { //否则每次+1
                            emit('update:cur_focus_app', props.cur_focus_app + 1);
                        }
                    } else { //否则直接每次+8
                        emit('update:cur_focus_app', props.cur_focus_app + 8);
                    }
                }
            }
            break;
        case 'left':
            {
                if (props.cur_focus_app > 0) {
                    emit('update:cur_focus_app', props.cur_focus_app - 1);
                } else {
                    emit('update:cur_focus_app', all_length - 1);
                }
            }
            break;
        case 'right':
            {
                if (props.cur_focus_app < all_length - 1) {
                    emit('update:cur_focus_app', props.cur_focus_app + 1);
                } else {
                    emit('update:cur_focus_app', 0);
                }
            }
            break;
    }
    await nextTick();
    let act_app = document.querySelector('.active');
    act_app?.scrollIntoView({
        block: 'nearest',
        inline: 'nearest',
        behavior: 'smooth'
    });
    return false;
}

function fun_expand() {
    if (search_result_is_expand.value) {
        search_result_is_expand.value = false;
    } else {
        search_result_is_expand.value = true;
    }
    search();
    adjust_height();
}

function adjust_height() {
    //主搜索框高度
    let search_box_height = 50;
    //推荐栏高度
    let recommand_height = recommend_list.length == 0 ? 0 : 120;
    //搜索结果高度
    let search_resule_height = 120;
    if (search_result_is_expand.value) {
        search_resule_height += 90 * Math.floor((search_result_list.length - 1) / 8);
    }
    if (search_result_list.length == 0) {
        search_resule_height = 0
    }

    if (search_resule_height + recommand_height > 550) {
        search_resule_height = 430;
    }
    // console.log(search_box_height + search_resule_height + recommand_height);
    Window.set_height(search_box_height + search_resule_height + recommand_height);
}

async function fun_open_app(app: AppInfo, is_from_search: boolean) {
    const app_list = applistStore.applist;
    for (let i = 0; i < app_list.length; i++) {
        if (app_list[i].name == app.name) {
            app_list[i].weight += 1;
            break;
        }
    }
    let appinfo = [];
    for (let i = 0; i < app_list.length; i++) {
        appinfo.push({
            name: app_list[i].name,
            weight: app_list[i].weight,
        });
    }
    //最多领先后一个程序3次，防止被某个程序始终占据最前面的位置
    appinfo.sort((a, b) => {
        return b.weight - a.weight;
    })
    for (let i = appinfo.length - 1; i > 0; i--) {
        if (appinfo[i - 1].weight - appinfo[i].weight > 3) {
            appinfo[i - 1].weight = appinfo[i].weight + 3;
        }
    }
    configStore.write_appinfo(appinfo);

    if (is_from_search) { //如果是搜索进入，则需要清空features
        Ombra.set_features([]);
    }

    Ombra.set_appid(app.id);
    app.setup();
    if (app.component == null) {//无界面，则隐藏窗口
        Window.hide();
    } else {  //有界面，则跳转到app页面
        Window.to_app(app.id);
    }
}
let old_search_content = "";
//启动搜索
async function search(features = []) {
    const app_list = applistStore.applist;

    let search_content = Ombra.get_text();

    //非扩展状态、且搜索内容变化，则重新折叠面板
    if (old_search_content != search_content) {
        old_search_content = search_content;
        search_result_is_expand.value = false;
    }

    if (features.length == 0) { //如果未传入匹配特性，则自行匹配添加
        //首先根据输入内容匹配特性
        let fe = await match_feature(search_content);
        Ombra.set_features(fe);
    } else { //否则，直接设置传入的特性
        Ombra.set_features(features);
    }

    //用于临时存储匹配结果
    let tmp_match_result = [];
    //用于临时存储推荐结果
    let tmp_recommend_result = [];

    for (let app_item of app_list) {

        //匹配推荐应用
        const features_list = Ombra.get_features();
        for (let f of app_item.feature) {
            if (features_list.includes(f)) {
                let app = await test_name_match(app_item);
                tmp_recommend_result.push(app);
            }
        }

        //跳过only_featur显示
        if (app_item.only_feature) continue;

        //匹配搜索应用
        let app = await test_name_match(app_item, search_content);
        if (app.is_match) {
            tmp_match_result.push(app);
        }
    }
    //根据权重排序
    tmp_recommend_result.sort((a, b) => {
        return b.weight - a.weight;
    })
    recommend_list.length = 0; //清空
    //最多8个推荐应用
    recommend_list.push(...tmp_recommend_result.slice(0, 8));
    //根据权重排序
    tmp_match_result.sort((a, b) => {
        return b.weight - a.weight;
    })
    search_result_list.length = 0; //清空
    //当前是否展开
    if (search_result_is_expand.value) {
        search_result_list.push(...tmp_match_result);
    } else { //在未展开模式下，最多显示8个应用
        search_result_list.push(...tmp_match_result.slice(0, 8));
    }

    //显示的搜索结果始终为实际匹配到的个数
    search_count.value = tmp_match_result.length;
    //调整窗口
    adjust_height();
    return;
}

//根据app name匹配构造要用于显示的show_name标签内容
async function test_name_match(app: AppInfo, search = ''): Promise<AppInfoExt> {
    const appName = app.name;
    const lowerAppName = appName.toLowerCase();
    const lowerSearch = search.toLowerCase();

    //没有搜索内容，则直接返回
    if (search.length == 0) {
        return {
            is_match: true,
            show_name: Tools.get_span(appName, 'normal'),
            ...app
        };
    }

    //直接搜索
    let pos = lowerAppName.indexOf(lowerSearch);
    if (pos != -1) {
        let s = Tools.get_span(appName.substring(0, pos), 'normal');
        s += Tools.get_span(appName.substring(pos, pos + search.length), 'match');
        s += Tools.get_span(appName.substring(pos + search.length), 'normal');
        const weight = app.weight + (pos >= 3 ? 0 : 3 - pos);
        return {
            is_match: true,
            show_name: s,
            ...app,
            weight: weight
        };
    }

    //如果搜索内容中没有汉字，则可以尝试英文首字母匹配、中文汉字拼音匹配、中文汉字首字母匹配
    if (!/.*[\u4e00-\u9fa5].*/.test(search)) {
        //如果appName为字母数字空格组成，即没有汉字，尝试英文首字母匹配
        if (/^[\w\s]+$/.test(appName)) {
            // 将appName先拆分为单词
            const words = appName.split(/\s+/).filter(word => word.length > 0);
            //获取所有单词首字母组成的序列
            const initials_str = words.map(word => word[0]).join('').toLowerCase();
            let pos = initials_str.indexOf(search.toLowerCase());
            if (pos != -1) { //如果匹配到了
                let s = '';
                for (let i = 0; i < words.length; i++) {
                    if (i >= pos && i < pos + search.length) {
                        s += Tools.get_span(words[i][0], 'match');
                        s += Tools.get_span(words[i].substring(1) + ' ', 'normal');
                    } else {
                        s += Tools.get_span(words[i] + ' ', 'normal');
                    }
                }

                const weight = app.weight + (pos >= 3 ? 0 : 3 - pos);

                return {
                    is_match: true,
                    show_name: s,
                    ...app,
                    weight: weight,
                };
            }
        }
        //如果appName中包含中文，则尝试单独汉字拼音匹配
        if (/.*[\u4e00-\u9fa5].*/.test(appName)) {
            for (let index = 0; index < appName.length; index++) {
                let c = appName.charAt(index);
                //如果某个字符为汉字
                if (/[\u4e00-\u9fa5]/.test(c)) {
                    let py = await Ombra.to_pinyin(c); //将其转化为拼音
                    //如果该汉字的拼音以搜索的字符串作为开头，则表示匹配成功，否则匹配失败
                    if (py[0].indexOf(lowerSearch) != 0) continue;
                    let s = Tools.get_span(appName.substring(0, index), 'normal');
                    s += Tools.get_span(appName.substring(index, index + 1), 'match');
                    s += Tools.get_span(appName.substring(index + 1), 'normal');

                    const weight = app.weight + (pos >= 3 ? 0 : 3 - pos);

                    return {
                        is_match: true,
                        show_name: s,
                        ...app,
                        weight: weight
                    };
                }
            }
        }
        //如果appName由汉字开头，则尝试拼音首字母匹配
        if (/^[\u4e00-\u9fa5]+.*$/.test(appName)) {
            let han_pos = appName.length;
            for (let i = 0; i < appName.length; i++) {
                let c = appName.charAt(i);
                if (!/[\u4e00-\u9fa5]/.test(c)) {
                    han_pos = i;
                    break;
                }
            }
            let hans = appName.substring(0, han_pos);
            let pys = await Ombra.to_pinyin(hans); //将汉字转化为拼音
            //获取所有拼音首字母组成的序列
            const initials_str = pys.map(py => py[0]).join('').toLowerCase();
            let pos = initials_str.indexOf(search.toLowerCase());
            let s = '';
            if (pos != -1) { //如果匹配到了
                s += Tools.get_span(appName.substring(0, pos), 'normal');
                s += Tools.get_span(appName.substring(pos, pos + search.length), 'match');
                s += Tools.get_span(appName.substring(pos + search.length), 'normal');
                let weight = app.weight;
                if (pos == 0) { //if in header
                    weight += 3;
                }
                return {
                    is_match: true,
                    show_name: s,
                    ...app,
                    weight: weight,
                };
            }
        }
    }

    //没有匹配项的返回结果
    return {
        is_match: false,
        show_name: Tools.get_span(appName, 'normal'),
        ...app
    };
}

//根据搜索内容返回可能的特性
async function match_feature(cnt: string) {
    let tmp_feature = [] as Array<string>
    if (cnt.length == 0) return tmp_feature;
    tmp_feature.push('text');
    let re_sep = /^[^\/\\:\*\?"<>|]*$/g;
    if (re_sep.test(cnt)) {
        tmp_feature.push('filename');
    }
    var re = /^(?:http(s)?:\/\/)?[\w.-]+(?:\.[\w\.-]+)+[\w\-\._~:/?#[\]@!\$&'\*\+,;=.]+$/g;
    if (re.test(cnt)) {
        tmp_feature.push('url');
    }

    let r_path = /^[a-zA-Z]:\\([^\/\\:\*\?"<>|]+\\)*[^\/\\:\*\?"<>|]*\\?$/
    if (r_path.test(cnt)) {
        if (await FS.is_dir(cnt)) {
            tmp_feature.push('dir_path');
        } else {
            tmp_feature.push('file_path');
        }
    }
    return tmp_feature;
}

function recommand_item_is_active(index: number) {
    if (search_result_list.length != 0) {
        return index == props.cur_focus_app - search_result_list.length;
    } else {
        return index == props.cur_focus_app;
    }
}

defineExpose({
    click_app,
    move,
    search
});

</script>

<template>
    <div class="AppMenu" @click="props.main_input.focus()">
        <div class="SearchResult" v-if="search_result_list.length != 0">
            <div class="title" @click="fun_expand">
                <span>查找工具</span>
                <span>
                    <span>{{ search_result_is_expand ? '折叠' : '展开' }}</span>
                    <span>({{ search_count }})</span>
                </span>
            </div>
            <div class="items">
                <div v-for="(item, index) in search_result_list" class="item"
                    :class="{ 'active': index == props.cur_focus_app }" @click="fun_open_app(item, true)">
                    <img :src="item.icon" draggable="false">
                    <div class="name" v-html="item.show_name"></div>
                </div>
            </div>
        </div>
        <div class="RecommandResult" v-if="recommend_list.length != 0">
            <div class="title">
                <span>推荐工具</span>
            </div>
            <div class="items">
                <div v-for="(item, index) in recommend_list" class="item"
                    :class="{ 'active': recommand_item_is_active(index) }" @click="fun_open_app(item, false)">
                    <img :src="item.icon" draggable="false">
                    <div class="name"> <span>{{ item.name }}</span> </div>
                </div>
            </div>
        </div>
    </div>
</template>

<style scoped lang="less">
.AppMenu {
    background-color: #252525;
    display: flex;
    flex-direction: column;

    .SearchResult {
        height: 120px;
        flex-grow: 1;
        display: flex;
        flex-direction: column;

        .title {
            color: #cbcbcc;
            font-weight: bold;
            padding: 0 10px;
            font-size: 15px;
            height: 30px;
            line-height: 30px;
            user-select: none;
            cursor: pointer;

            &:hover {
                background: #3b3c3d;
            }

            display: flex;
            justify-content: space-between;
        }

        .items {
            display: flex;
            flex-wrap: wrap;
            background-color: #252525;
            overflow: auto;
            height: 90px;
            flex-grow: 1;

            &::-webkit-scrollbar {
                width: 5px;
            }

            &::-webkit-scrollbar-thumb {
                background-color: #434343;
                border-radius: 3px;
            }

            &::-webkit-scrollbar-track {
                background-color: #252525;
            }

            .item {
                height: 90px;
                width: 99px;
                display: flex;
                flex-direction: column;
                justify-content: center;
                align-items: center;
                cursor: pointer;
                user-select: none;
                padding: 10px;
                text-align: center;

                &:hover {
                    background-color: #3c3c3e;
                }

                img {
                    width: 30px;
                    height: 30px;
                    border-radius: 5px;
                    user-select: none;
                }

                .name {
                    font-size: 12px;
                    user-select: none;
                    height: 35px;
                    margin: 7px 4px 0;

                    display: -webkit-box;
                    -webkit-box-orient: vertical;
                    -webkit-line-clamp: 2;
                    overflow: hidden;
                    color: #d2d2d2;
                    word-break: break-all;

                    &:deep(.normal) {
                        color: #d2d2d2;
                    }

                    &:deep(.match) {
                        color: #f18325;
                    }
                }
            }

            .active {
                background-color: #565656;
            }
        }
    }

    .RecommandResult {
        height: 120px;

        .title {
            color: #cbcbcc;
            font-weight: bold;
            padding: 0 10px;
            font-size: 15px;
            height: 30px;
            line-height: 30px;
            user-select: none;
        }

        .items {
            display: flex;
            flex-wrap: wrap;
            background-color: #252525;
            overflow: hidden;
            height: 90px;
            flex-grow: 1;

            .item {
                height: 90px;
                width: 99px;
                display: flex;
                flex-direction: column;
                justify-content: center;
                align-items: center;
                cursor: pointer;
                user-select: none;
                padding: 10px;

                &:hover {
                    background-color: #3c3c3e;
                }

                img {
                    width: 30px;
                    height: 30px;
                    border-radius: 5px;
                    user-select: none;
                }

                .name {
                    font-size: 12px;
                    user-select: none;
                    height: 35px;
                    margin: 7px 4px 0;
                    text-align: center;

                    word-break: break-all;
                    text-overflow: ellipsis;
                    display: -webkit-box;
                    -webkit-box-orient: vertical;
                    -webkit-line-clamp: 2;
                    /* 这里是超出几行省略 */
                    overflow: hidden;
                    color: #d2d2d2;
                }
            }

            .active {
                background-color: #565656;
            }
        }
    }

    .plugin {
        height: 200px;
    }
}
</style>