/*
 * Copyright 2014-2020 the original author or authors.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
<template>
    <div style="height:100%">
        <content-head :headTitle="$t('title.transactionAudit')" :headSubTitle="$t('title.unusualUser')" @changGroup="changGroup"></content-head>
        <div class="module-wrapper" style="position: relative;">
            <div class="search-part">
                <div class="search-part-left">
                    <el-tooltip effect="dark" :content="$t('transaction.unusualTips')" placement="top-start">
                        <i class="el-icon-info contract-icon font-15">Tips</i>
                    </el-tooltip>
                    <!-- <el-radio-group v-model="type" @change="changeUserList">
                        <el-radio :label="item.value" :key='item.value' v-for="item in userOptions">{{item.name}}</el-radio>
                    </el-radio-group> -->
                    <!-- <el-select :placeholder="'切换选择不同类型的用户'" v-model="type"  @change="changeUserList">
                        <el-option
                        v-for="item in userOptions"
                        :key="item.value"
                        :label="item.name"
                        :value="item.value">
                        </el-option>
                    </el-select> -->
                </div>

                <div class="search-part-right">
                    <el-input :placeholder="$t('transaction.inputUserAddress')" v-model="userName" class="input-with-select" clearable @clear="clearText">
                        <el-button slot="append" icon="el-icon-search" @click="search"></el-button>
                    </el-input>
                </div>
            </div>
            <div class="search-table">
                <el-table :data="hashData" tooltip-effect="light" style="overflow-y: auto; height:100%" v-loading="loading">
                    <el-table-column type="expand">
                        <template slot-scope="props">
                            <div class="expand-item">
                                <ul class="expand-item-ul">
                                    <li v-for="item in props.row['hashs']" :key='item.hash'>
                                        <div @click="showItem(item)" class="expand-item-div">
                                            <i :class="item.show?'el-icon-arrow-down':'el-icon-arrow-up'"></i>
                                            <span class="expand-item-span">TxHash：
                                                <span>{{item.hash}}</span>
                                            </span>
                                        </div>
                                        <transaction-detail :transHash="item.hash" v-if="item.show"></transaction-detail>
                                    </li>
                                </ul>
                            </div>
                        </template>
                    </el-table-column>
                    <el-table-column v-for="head in unusualUserHead" :label="head.name" :key="head.enName" show-overflow-tooltip align="center" v-if="head.enName!=='hashs'">
                        <template slot-scope="scope">
                            <span>{{scope.row[head.enName]}}</span>
                        </template>
                    </el-table-column>
                    <el-table-column  :label="$t('nodes.operation')" width="200">
                        <template slot-scope="scope">
                            <el-button v-if="!(disabled || scope.row.userId > 0)"   @click="importData(scope.row)" type="text">{{$t("system.import")}}</el-button>
                            <el-button v-if="disabled || scope.row.userId > 0"  @click="routeLink(scope.row)" type="text">{{$t("text.checkData")}}</el-button>
                        </template>
                    </el-table-column>
                </el-table>
                <el-pagination class="page" @size-change="handleSizeChange" @current-change="handleCurrentChange" :current-page="currentPage" :page-sizes="[10, 20, 30, 50]" :page-size="pageSize" layout="total, sizes, prev, pager, next, jumper" :total="total">
                </el-pagination>
            </div>
        </div>
        <el-dialog :visible.sync="$store.state.creatUserVisible" :title="$t('privateKey.createUser')" width="640px" :append-to-body="true" class="dialog-wrapper" v-if='$store.state.creatUserVisible' center>
            <v-creatUser @creatUserClose="creatUserClose"  @bindUserClose="bindUserClose" ref="creatUser" :address='address'></v-creatUser>
        </el-dialog>
    </div>
</template>

<script>
import { unusualUserList, getAllUserList } from "@/util/api";
import contentHead from "@/components/contentHead";
import transactionDetail from "@/components/transactionDetail";
import creatUser from "../privateKeyManagement/components/creatUser.vue";
export default {
    name: "unusualUser",
    components: {
        contentHead,
        transactionDetail,
        "v-creatUser": creatUser,
    },
    data() {
        return {
            disabled: false,
            loading: false,
            noticeVisibility: false,
            currentPage: 1,
            pageSize: 10,
            total: 0,
            userName: '',
            unusualUserList: [],
            userOptions: [
                {
                    name: this.$t('text.totalUser'),
                    value: 1
                },
                {
                    name: this.$t('text.normalUser'),
                    value: 2
                },
                {
                    name: this.$t('text.unregisteredUser'),
                    value: 3
                }
            ],
            type: 1
        };
    },
    computed: {
        unusualUserHead() {
            let data = [
                {
                    enName: "address",
                    name: this.$t('privateKey.userAddress')
                },
                {
                    enName: "userName",
                    name: this.$t('privateKey.userName')
                },
                {
                    enName: "transCount",
                    name: this.$t('home.chartTransactions')
                },
                {
                    enName: "modifyTime",
                    name: this.$t('transaction.transactionTime')
                },
                {
                    enName: "hashs",
                    name: this.$t('transaction.hash')
                }
            ]
            return data
        },
        hashData() {
            var arr = this.unusualUserList,
                list = [];
            arr.forEach(item => {
                if (item.hashs) {
                    item.hashs = item["hashs"].split(",");
                }
            });
            arr.forEach(item => {
                let hashArr = [];
                if (item.hashs) {
                    for (let i = 0; i < item.hashs.length; i++) {
                    let obj = {};
                    if (i === 0) {
                        obj.hash = item.hashs[i];
                        obj.show = true;
                    } else {
                        obj.hash = item.hashs[i];
                        obj.show = false;
                    }
                    hashArr.push(obj);
                }
                item.hashs = hashArr;
                }
            });
            return arr;
        }
    },
    mounted() {
        if ((localStorage.getItem("root") === "admin" || localStorage.getItem("root") === "developer") && localStorage.getItem("groupId")) {
            this.disabled = false
        } else {
            this.disabled = true
        }
        if (localStorage.getItem("groupId") && (localStorage.getItem("configData") == 3 || localStorage.getItem("deployType") == 0)) {
            this.getUnusualUserList();
        }
    },
    methods: {
        changeUserList(val) {
            console.log(val)
            this.type = val
            this.getUnusualUserList()
        },
        changGroup() {
            this.getUnusualUserList()
        },
        closeNotice() {
            this.noticeVisibility = false;
        },
        getUnusualUserList() {
            this.loading = true;
            let groupId = localStorage.getItem("groupId");
            let reqData = {
                groupId: groupId,
                pageNumber: this.currentPage,
                pageSize: this.pageSize
            },
                reqQuery = {
                    type: 3
                };

            if (this.userName) {
                reqQuery['address'] = this.userName;
            }
            getAllUserList(reqData, reqQuery)
                .then(res => {
                    if (res.data.code === 0) {
                        this.unusualUserList = res.data.data;
                        this.total = res.data.totalCount;
                        this.loading = false;
                    } else {
                        this.loading = false;
                        this.$message({
                            message: this.$chooseLang(res.data.code),
                            type: "error",
                            duration: 2000
                        });
                    }
                })
                .catch(err => {
                    this.loading = false;
                    this.$message({
                        message: err.data || this.$t('text.systemError'),
                        type: "error",
                        duration: 2000
                    });
                });
        },
        hashList(val) {
            let list = [];
            list = val.split(",");
            return list;
        },
        showItem(val) {
            this.$set(val, "show", !val.show);
        },
        search() {
            this.currentPage = 1
            this.getUnusualUserList();
        },
        handleSizeChange: function (val) {
            this.pageSize = val;
            this.currentPage = 1;
            this.getUnusualUserList();
        },
        handleCurrentChange: function (val) {
            this.currentPage = val;
            this.getUnusualUserList();
        },
        clearText: function () {
            this.getUnusualUserList()
        },
        importData(val) {
            this.address = val.address;
            this.$store.dispatch('switch_creat_user_dialog')
        },
        creatUserClose () {
            this.getUnusualUserList();
        },
        bindUserClose () {
            this.getUnusualUserList();
        },
        routeLink(val) {
            this.$router.push({
                path: '/privateKeyManagement',
                query: {
                    userParam: val.address
                }
            })
        }
    }
};
</script>

<style scoped>
.search-part-right {
    float: right;
    width: 464px;
}
.search-part-left {
    float: left;
    height: 40px;
    line-height: 40px;
}
.input-with-select >>> .el-input__inner {
    border-top-left-radius: 20px;
    border-bottom-left-radius: 20px;
    border: 1px solid #eaedf3;
    box-shadow: 0 3px 11px 0 rgba(159, 166, 189, 0.11);
}
.input-with-select >>> .el-input-group__append {
    border-top-right-radius: 20px;
    border-bottom-right-radius: 20px;
    box-shadow: 0 3px 11px 0 rgba(159, 166, 189, 0.11);
}
.input-with-select >>> .el-button {
    border: 1px solid #20d4d9;
    border-radius: inherit;
    background: #20d4d9;
    color: #fff;
}

.search-table >>> .el-table__expanded-cell[class*="cell"] {
    padding-left: 20px;
    padding-top: 0;
}
.search-table >>> .el-table__body-wrapper {
    overflow-y: auto;
}

.expand-item-div {
    display: block;
    cursor: context-menu;
    padding: 17px 0;
    border-bottom: 1px dashed #e8e8e8;
    border-top: 1px dashed #e8e8e8;
    margin-bottom: 5px;
    padding-left: 21px;
}

.expand-item-div:first-child {
    border-top: none;
}

.expand-item-div:hover {
    background: #f3f5f9;
}
.expand-item-span {
    margin-left: 26px;
    font-size: 12px;
    color: #00122c;
}
.expand-item-span > span {
    color: #515356;
    margin-left: 47px;
}
.notify-cation {
    margin: 0 auto;
    width: 300px;
    position: relative;
    bottom: 50%;
    box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.1);
    padding: 14px 24px 14px 13px;
    border: 1px solid #ebeef5;
    border-radius: 8px;
    z-index: 1000;
    padding-left: 30px;
    background: #fff;
}
.notify-cation > div {
    text-align: right;
    cursor: pointer;
}
</style>
