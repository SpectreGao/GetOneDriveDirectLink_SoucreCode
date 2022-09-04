<template>
    <div class="box">
        <div class="card">
            <h3>
                <Icon icon="Microsoft" size="24" color="#488ded" />控制面板
            </h3>
            <div>
                <button @click="launchOneDrivePicker('query')">
                    <Icon icon="Cloud" />从OneDrive选择文件
                </button>
                <button @click="launchOneDrivePicker('share')">
                    <Icon icon="Share" />额外创建分享链接（配合1drv.ws使用）
                </button>
            </div>
            <ul class="mask">
                <li
                    v-for="(item,index) in maskCfg"
                    :key="index"
                    :class="selectIndex == index ? 'active':''"
                    @click="selectChange(index)"
                >{{item.name}}</li>
            </ul>
            <div class="mask">
                <p>{{maskCfg[selectIndex].cfg}}</p>
            </div>
        </div>
        <div class="card out">
            <h3>
                <Icon icon="Microsoft" size="24" color="#488ded" />输出面板
            </h3>
            <div class="tip">{{msg}}</div>
            <ul class="mask">
                <li v-for="(item,index) in result" :key="index" @click="copylink(item)">{{item}}</li>
            </ul>
            <button @click="copylink(result)" v-if="result.length">全部复制</button>
            <div class="copyright">
                Copyright © 2017
                <a
                    href="https://github.com/Mapaler/GetOneDriveDirectLink"
                    target="_blank"
                >枫谷剑仙</a>
                版权所有，本程序源代码使用GPLv3协议公开
                <br />Mapaler all right reserved. Source
                public
                by GPLv3.
                <div class="hr"></div>
                <p>
                    使用 Vue3 进行二次开发与美化 by 2022
                    <a
                        href="https://github.com/SpectreGao/GetOneDriveDirectLink_Code"
                        target="_blank"
                    >SpectreGao</a>，如有侵权，将第一时间删除
                </p>
            </div>
        </div>
    </div>
</template>
  

<script setup>
import { ref, onMounted } from "vue";
import Icon from "./components/Icon.vue";

const selectIndex = ref(0);
const msg = ref("点击按钮从OneDrive选择文件");
const result = ref([]);
const redata = ref(null);

const launchOneDrivePicker = (action = "query") => {
    if (action == "query") {
        msg.value = "正在等待API返回数据";
        Alert("正在等待API返回数据");
        let odOptions = {
            clientId: "5712d2c8-2c32-4f4f-a5aa-fdc966092171",
            action: action, //share | download | query
            multiSelect: true,
            openInNewWindow: true,
            //advanced: {createLinkParameters: { type: "embed", scope: "anonymous" }},
            advanced: {
                queryParameters:
                    "select=audio,content,createdBy,createdDateTime,cTag,deleted,description,eTag,file,fileSystemInfo,folder,id,image,lastModifiedBy,lastModifiedDateTime,location,malware,name,package,parentReference,photo,publication,remoteItem,root,searchResult,shared,sharepointIds,size,specialFolder,video,webDavUrl,webUrl,activities,children,listItem,permissions,thumbnails,versions,@microsoft.graph.conflictBehavior,@microsoft.graph.downloadUrl,@microsoft.graph.sourceUrl"
            },
            success: function(files) {
                redata.value = files; //存入全局数组
                console.log(
                    "本次返回 %d 个文件，数据为 %o",
                    redata.value.value.length,
                    redata.value
                );
                generate_output(redata.value);
            },
            cancel: () => {
                msg.value = "取消操作";
                Alert("取消操作");
            },
            error: function(e) {
                msg.value = "发生错误";
                Alert("发生错误");
                result.value = e.toString();
            }
        };
        OneDrive.open(odOptions);
    }
};

const generate_output = files => {
    let mask = maskCfg[selectIndex.value];
    let filearr = files.value;

    msg.value = "共选择 " + filearr.length + " 个文件。";
    Alert("共选择 " + filearr.length + " 个文件。");
    if (
        filearr.some(function(item) {
            return item.shared == undefined || item.shared.scope != "anonymous";
        })
    ) {
        msg.value += "存在非公共权限文件，注意添加通行许可代码。";
    }

    let outStrArr = filearr.map(function(item, index) {
        let outStr = showMask(mask.cfg, item, index);
        return outStr;
    });
    result.value = outStrArr;
};

//显示掩码用
const showMask = (str, file, index) => {
    var newTxt = eval("`" + str + "`");
    var pattern = "%{([^}]+)}";
    var rs = null;

    while ((rs = new RegExp(pattern).exec(newTxt)) != null) {
        var mskO = rs[0], //包含括号的原始掩码
            mskN = rs[1]; //去掉掩码括号
        if (mskN != undefined) {
            mskN =
                mskN != undefined
                    ? mskN
                          .replace(/\\{/gi, "{")
                          .replace(/\\}/gi, "}")
                          .replace(/\\\\/gi, "\\")
                    : null;
            try {
                var evTemp = eval(mskN);
                if (evTemp != undefined)
                    newTxt = newTxt.replace(mskO, evTemp.toString());
                else newTxt = newTxt.replace(mskO, "");
            } catch (e) {
                msg.value = "掩码异常，详情查看控制台";
                Alert("掩码异常，详情查看控制台");
                console.error(mskO + " 掩码出现了异常情况", e);
            }
        }
    }

    return newTxt;
};

const selectChange = index => {
    selectIndex.value = index;
    if (redata.value) generate_output(redata.value);
};

const maskCfg = [
    {
        name: "普通外链",
        cfg: "http://storage.live.com/items/${file.id}:/${file.name}"
    },
    {
        name: "最短链接",
        cfg: "http://storage.live.com/items/${file.id}"
    },
    {
        name: "UBB代码外链图片",
        cfg: "[img]http://storage.live.com/items/${file.id}:/${file.name}[/img]"
    },
    {
        name: "模板字符串基本使用示例",
        cfg: "在OneDrive里查看 ${file.name} 的地址是：${file.webUrl}"
    },
    {
        name: "表达式使用示例",
        cfg:
            '${index+1}号文件的尺寸是：${file.size>1024?Math.round(file.size/1024)+"K":file.size}B'
    },
    {
        name: "自动选择img/mp3 UBB代码",
        cfg:
            '[${file.image?"img":(file.audio?"mp3":"file")}]http://storage.live.com/items/${file.id}:/${file.name}[/${file.image?"img":(file.audio?"mp3":"file")}]'
    },
    {
        name: "ES6完整文件尺寸换算示例",
        cfg:
            '${index+1}号文件的尺寸是：${(function(size){const bArr = ["B","KiB","MiB","GiB","TiB"];for(let idx=0;idx<bArr.length;idx++){if(idx<bArr.length && size/Math.pow(1024,idx+1)>1)continue;else return (size/Math.pow(1024,idx)).toFixed(2) + " " + bArr[idx];}})(file.size)}'
    },
    {
        name: "ES6闭包函数示例1",
        cfg:
            '文件的权限是：${(scope=>{switch(scope){case "anonymous":return "所有人";case "users":return "仅限指定用户";default:return "私有";}})(file.shared?file.shared.scope:null)}'
    },
    {
        name: "ES6闭包函数示例2",
        cfg:
            '文件年份：${(createTime=>new Date(createTime).toLocaleString(\'zh-u-ca-chinese-nu-hanidec\',{year:"numeric",month:"long"}))(file.createdDateTime)}'
    },
    // {
    //     name: "第三方 1drv.ws 项目",
    //     cfg: '${file.permissions[0].link.webUrl.replace("1drv.ms","1drv.ws")}'
    // },
    {
        name: "官方图片下载直连（短期？）",
        cfg:
            '${file["@microsoft.graph.downloadUrl"].replace(/public.w+.files/i,"public.ch.files")}'
    }
];

const copylink = item => {
    let contents = "";
    let copyInput;
    if (typeof item == "string") {
        copyInput = document.createElement("input");
        document.body.appendChild(copyInput);
        copyInput.setAttribute("value", item);
    } else {
        for (var i = 0; i < item.length; i++) {
            contents += item[i] + "\n";
        }
        copyInput = document.createElement("textarea");
        document.body.appendChild(copyInput);
        copyInput.value = contents;
    }
    copyInput.select();
    document.execCommand("copy");
    Alert("链接已复制到剪贴板！");
    copyInput.remove();
};

onMounted(() => {
    if (
        location.protocol != "https:" &&
        location.hostname != "localhost" &&
        location.hostname != ""
    ) {
        var goto = confirm(
            "检测到你正在使用http模式，本应用要求使用https模式。\n是否自动跳转？"
        );
        if (goto) {
            location.protocol = "https:";
        }
    }
});

const Alert = info => {
    if (!$("#SpectreAlert").length > 0)
        $(
            `<div id="SpectreAlert"><style>#SpectreAlert {position: fixed;z-index: 99999;top: 20px;right: 20px}#SpectreAlert>div {width: 350px !important;height: auto;display: none;padding: 10px 30px;background: rgba(255, 255, 255, .7);backdrop-filter: blur(5px);border: 1px solid #ddd;border-radius: 4px;box-shadow: 0 0 5px #ccc;margin-bottom: 12px;text-align: center;color: #555;position: relative;font-size: 13px;cursor: pointer;white-space: normal !important;text-overflow: clip !important;overflow: visible !important;animation: SMP_Show .8s;}@keyframes SMP_Show {0% {opacity: 0;transform: scale(.8) translateX(100%);}80%{transform: scale(.95) translateX(0%);}100% {opacity: 1;transform: scale(1) translateX(0%);}}.SpectreAlertClose {position: absolute;top: 50%;transform: translateY(-50%);right: 10px;padding: 10px 2px;transition: .3s}.SpectreAlertClose:hover {background: #eee;border-radius: 3px;}.SpectreAlertClose:active {background: #ddd;}.SpectreAlertClose::before,.SpectreAlertClose::after {content: '';width: 14px;height: 1px;background: gray;display: block}.SpectreAlertClose::before {transform: rotate(45deg)}.SpectreAlertClose::after {transform: translateY(-1px) rotate(-45deg)}</style></div>`
        ).appendTo(document.body);
    let time = new Date().toLocaleTimeString();

    $(`<div>[${time}] ${info}<span class="SpectreAlertClose"></span></div>`)
        .prependTo("#SpectreAlert")
        .fadeToggle("slow", function() {
            $(".SpectreAlertClose").click(function() {
                $(this)
                    .parent()
                    .fadeToggle("fast", function() {
                        $(this).remove();
                    });
            });
            setTimeout(() => {
                $(this).fadeToggle("slow", function() {
                    $(this).remove();
                });
            }, 3000);
        });
};
</script>
