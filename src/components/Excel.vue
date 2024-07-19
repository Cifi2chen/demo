<script setup>
import { ref,watchEffect } from 'vue'

//模拟维度数据
let dimensionsMock = {
    A: ["a1","a2","a3",{text:"a4",childs:["a41",{text:"a42",childs:["a421"]},"a43"]}],
    B: ["b1","b2"],
    C: ["c1","c2","c3"],
    D: ["d1",{text: "d2",childs:["d21","d22"]}],
    E: ["e1","e2"]
};

//模拟行头配置数据
// test 分支增加了注释内容
let xConfigMock = {
    operator: "*",
    elements: [{
        operator: "+",
        elements: ["B","D"]
    },"C"]
};
// 维度成员转换为组合配置
function dimToConfig(ts){
    let el = {
        operator: "+",
        elements: []
    }
    for(let i=0;i<ts.length;i++){
        if(ts[i].childs){
            el.elements.push({
                operator: "*",
                elements: [ts[i].text,dimToConfig(ts[i].childs)]
            })
        } else {
            el.elements.push(ts[i]);
        }
    }
    return el;
}

//挂载维度成员到组合配置
function preHookDim(cf){
    let ts = cf.elements;
    for(let i=0;i<ts.length;i++){
        let t = ts[i];
        if(t.elements){
            preHookDim(t);
        } else {
            cf.elements[i] = dimToConfig(dimensionsMock[t]);
        }
    }
}
// 计算width 和 heightc
function preCalcWH(el){
    if(el.operator == "*"){
        let totW = 1,totH = 0,ts = el.elements;
        for(let i=ts.length-1;i>=0;i--){
            let t = ts[i];
            if(!t.operator){
                el.elements[i] = {
                    text: t,
                    power: totW
                }
                // totW *= 1;
                totH += 1;
            } else {
                ts[i].power = totW;
                let pt = preCalcWH(t);
                totW *= pt.width;
                totH += pt.height; 
            }
        }
        el.width = totW;
        el.height = totH;
        return el;
    }
    if(el.operator == "+"){
        let totW = 0,totH = 1,ts = el.elements;
        for(let i=0;i<ts.length;i++){
            let t = ts[i];
            if(!t.operator){
                el.elements[i] = {
                    text: t
                }
                totW += 1;
            } else {
                let pt = preCalcWH(t);
                if(pt.height>totH){
                    totH = pt.height;
                }
                totW += pt.width;
            }
        }
        el.width = totW;
        el.height = totH;
        return el;
    }
}

//重新计算并修正height+power
function preCalcHPAgain(el){
    if(el.operator=="+"){
        let pH = el.height,pO = el.power,ts = el.elements;
        let start = el.start?el.start:1;
        for(let i=0;i<ts.length;i++){
            let t = ts[i];
            let tH = t.height?t.height:1;
            ts[i].start = start;
            if(pO){
                let tO = t.power?t.power:1;
                ts[i].power = pO*tO;
                start += t.width*pO*tO;
                ts[i].length = t.width*pO*tO;
            } else {
                start += t.width;
                ts[i].length = t.width;
            }
            if(pH>tH){
                ts[i].height = pH;
            }
            preCalcHPAgain(t);
        }
    }
    if(el.operator=="*"){
        let pH = el.height,pO = el.power,ts = el.elements;
        let totH = 0
        for(let i=0;i<ts.length;i++){
            let t = ts[i];
            let tH = t.height?t.height:1;
            totH += tH;
        }
        //乘积高度不够由最后一行补齐
        if(pH>totH){
            let lastH = ts[ts.length-1].height?ts[ts.length-1].height:1;
            ts[ts.length-1].height = lastH + pH - totH;
        }

        let start = el.start?el.start:1;
        let length = el.length?el.length:el.width;
        for(let i=0;i<ts.length;i++){
            let t = ts[i];
            ts[i].start = start;
            ts[i].length = length;
            if(pO){
                let tO = t.power?t.power:1;
                ts[i].power = pO*tO;
            }
            preCalcHPAgain(ts[i]);
        }
    }
}

function pushAll(t,f){
    t.push(...f);
}
function toTable(el,st,ed){
    let table = [];
    if(!el.operator){
        let td = {
            text: el.text
        };
        if(ed>st){
            td.cSpan = ed-st+1;
        }
        if(el.height&&el.height>1){
            td.rSpan = el.height;
        }
        table.push([td]);
    }
    if(el.operator == "+"){
        for(let i=0;i<el.height;i++){
            table.push([]);
        }
        let startF = 0,endF = 0,ts = el.elements;
        let ph = el.height;
        for(let i=0;true;i++){
            let t = ts[i%ts.length];
            let w = t.width?t.width:1;
            let p = t.power?t.power:1;
            let th = t.height?t.height:1;
            endF = startF + w*p;
            if(st<=endF){
                if(endF<ed){
                    let tb = toTable(t,st-startF,w*p);
                    for(let m=0;m<tb.length;m++){
                        pushAll(table[m],tb[m],ph-th);
                    }
                    st = endF+1;
                } else {
                    let tb = toTable(t,st-startF,ed-startF);
                    for(let m=0;m<tb.length;m++){
                        pushAll(table[m],tb[m],ph-th);
                    }
                    break;
                }
            }
            startF = endF;
        }
        return table;
    }
    if(el.operator == "*"){
        let ts = el.elements;
        for(let i=0;i<ts.length;i++){
            let t = ts[i];
            let tb = toTable(t,st,ed);
            for(let j=0;j<tb.length;j++){
                table.push(tb[j]);
            }
        }
    }
    return table;
}

preHookDim(xConfigMock);
preCalcWH(xConfigMock);
preCalcHPAgain(xConfigMock);
console.info(xConfigMock);
let start = ref(3),end = ref(10);
let old = toTable(xConfigMock,1,xConfigMock.width);
let oldTable = ref(old);

let tb,table,cols,rows;
watchEffect(()=>{
    tb = toTable(xConfigMock,start.value,end.value);
    table = ref(tb);
    cols = ref(xConfigMock.width);
    rows = ref(xConfigMock.height);
    console.info(tb);
})
function moveRight(){
    start.value += 1;
    end.value += 1;
}
function moverLeft(){
    start.value -= 1;
    end.value -= 1;
}
</script>
<template>
    <div style="text-align: left;">
        <div><span>开始：</span><input v-model.number="start"/></div>
        <div><span>结束：</span><input v-model.number="end"/></div>
        <button @click="moveRight">==></button>
        <button @click="moverLeft"><==</button>
        <table class="table-cls">
            <colgroup>
                <col style="width:0px;"/>
                <col v-for="n in cols" style="width:50px;"/>
            </colgroup>
            <tr>
                <td>&nbsp;</td>
                <td v-for="n in cols">{{ n }}</td>
            </tr>
            <tr v-for="tr in oldTable">
                    <td>&nbsp;</td>
                    <td v-for="td in tr" :colspan="td.cSpan" :rowspan="td.rSpan">{{ td.text }}</td>
            </tr>
        </table>
        <table class="table-cls">
            <colgroup>
                <col style="width:0px;"/>
                <col v-for="n in cols" style="width:50px;"/>
            </colgroup>
            <tr v-for="(tr,index) in table">
                    <td>&nbsp;</td>
                    <td v-if="index==0&&start>1" :colspan="start-1" :rowspan="rows"></td>
                    <td v-for="td in tr" :colspan="td.cSpan" :rowspan="td.rSpan">{{ td.text }}</td>
                    <td v-if="index==0&&cols>end" :colspan="cols-end" :rowspan="rows"></td>
            </tr>
        </table>
    </div>
</template>
<style scoped>
.msg-cls {
  color: #888;
}
.table-cls {
    margin-top: 20px;
    border-collapse: separate;
    border-spacing: 0;
    border-width: 0;
    table-layout: fixed;
    width: 0;
    outline-width: 0;
    cursor: default;
    max-width: none;
    max-height: none;
    text-align: center;
    border-top: 1px solid #767676;
    border-left: 1px solid #767676;
}
.table-cls td{
    border-style: solid;
    border-top-width: 0;
    border-left-width: 0;
    border-right: 1px solid #767676;
    border-bottom: 1px solid #767676;
}
</style>
