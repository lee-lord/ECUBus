<template>
  <div>
    <el-dialog
      title="Edit Service"
      :visible.sync="cd1"
      width="80%"
      :close-on-press-escape="false"
      :before-close="editClose"
    >
      <AddService
        editService
        :mode="mode"
        :service="service"
        v-if="refresh"
        @edititem="editItem"
        ref="addService"
      />
    </el-dialog>
    <el-dialog title="User function" :visible.sync="cd" width="80%">
      <div style="text-align:center">
        <el-pagination
          layout="prev, pager, next"
          hide-on-single-page
          :total="codeNum*10"
          @current-change="showNewCode"
        ></el-pagination>
      </div>
      <div>function(writeData,readData){</div>
      <codemirror v-model="jsFn[codeIndex]" :options="codeOption" ref="cmEditor" />
      <div>}</div>
    </el-dialog>
    <el-table
      size="small"
      ref="basictable"
      row-key="date"
      border
      :data="doipTable"
      max-height="600px"
      style="width: 100%;text-align: center;"
      empty-text="No Service"
      :row-class-name="tableRowClassName"
      @selection-change="handleSelectionChange"
    >
      <el-table-column type="selection" width="55" align="center"></el-table-column>
      <el-table-column prop="service" label="Service Info" width="300">
        <template slot-scope="scope">
          <div
            v-if="scope.row.type=='uds'"
          >{{ scope.row.service.name}} (0X{{ scope.row.service.value.toString(16)}})</div>
          <div v-else>
            <el-tag size="mini">Group:</el-tag>
            <strong>{{ scope.row.service.name}}</strong>
          </div>
        </template>
      </el-table-column>
      <el-table-column label="Suppress" width="76" align="center">
        <template slot-scope="scope" v-if="scope.row.type=='uds'">
          <i class="el-icon-circle-check" v-if="scope.row.payload[0].suppress" style="color:green"></i>
          <i class="el-icon-circle-close" v-else style="color:red"></i>
        </template>
      </el-table-column>
      <el-table-column prop="payload" label="Payload">
        <template slot-scope="scope">
          <div v-if="scope.row.payload&&scope.row.type=='uds'">
            <div v-for="(item, key) in scope.row.payload" :key="key">
              <el-tag size="mini">{{item.name}}</el-tag>
              : {{item[item.name]}}
            </div>
          </div>
          <div
            v-else-if="scope.row.subtable&&scope.row.type=='group'"
            style="height:100px;overflow:auto"
          >
            <div v-for="(item, key) in scope.row.subtable" :key="key">
              <div v-if="item.payload">
                <div v-for="(subitem, subkey) in item.payload" :key="subkey">
                  <el-tag size="mini">{{key+'-'+subitem.name}}</el-tag>
                  : {{subitem[subitem.name]}}
                  <span v-if="subitem.type=='subfunction'">
                    <i class="el-icon-circle-check" v-if="subitem.suppress" style="color:green"></i>
                    <i class="el-icon-circle-close" v-else style="color:red"></i>
                  </span>
                </div>
              </div>
            </div>
          </div>
          <div v-else>NULL</div>
        </template>
      </el-table-column>

      <el-table-column prop="func" label="Script" width="100" align="center">
        <template slot-scope="scope">
          <el-button type="text" icon="el-icon-document" @click="showCode(scope.row)"></el-button>
        </template>
      </el-table-column>
      <el-table-column fixed="right" label="Action" width="150" align="center">
        <template slot="header">
          <el-button
            size="mini"
            :disabled="selectTable.length==0"
            @click="deleteMulti"
            icon="el-icon-delete"
            type="text"
          >Selected</el-button>
        </template>
        <template slot-scope="scope">
          <!-- <h1 v-if="scope.$index==tableErrorIndex">error</h1> -->
          <div v-if="scope.$index==tableErrorIndex-1" class="table_error">
            <i class="el-icon-error" @click="closeError"></i>
          </div>
          <el-button
            type="danger"
            size="mini"
            icon="el-icon-delete"
            circle
            @click="deleteService(scope.$index)"
            :disabled="running"
          ></el-button>
          <el-button
            type="primary"
            size="mini"
            icon="el-icon-edit-outline"
            circle
            @click="editService(scope.$index,scope.row)"
            :disabled="running"
          ></el-button>
        </template>
      </el-table-column>
    </el-table>
  </div>
</template>
<script>
import Sortable from "sortablejs";
import config from "./service.js";
import AddService from "./addservice.vue";
const { ipcRenderer } = require("electron");
import pregroup from "./predefgroup.js";
export default {
  components: {
    AddService
  },
  data: function() {
    return {
      refresh: false,
      sortable: {},
      cd: false,
      cd1: false,
      config: config["uds"],
      group: [],
      editIndex: 0,
      service: {},
      jsFn: [" "],
      codeIndex: 0,
      codeNum: 1,
      codeOption: {
        readOnly: true
      },
      selectTable: []
    };
  },
  mounted() {
    const table = document.querySelector(".el-table__body-wrapper tbody");
    const self = this;
    this.sortable = Sortable.create(table, {
      onEnd({ newIndex, oldIndex }) {
        if (self.mode === "can") {
          self.$store.commit("canTableUpdate", [newIndex, oldIndex]);
        } else if (self.mode === "doip") {
          self.$store.commit("doipTableUpdate", [newIndex, oldIndex]);
        } else if (self.mode === "lp") {
          self.$store.commit("lpTableUpdate", [newIndex, oldIndex]);
        } else {
          return;
        }
      }
    });
  },
  props: ["mode"],
  methods: {
    editClose(done) {
      var dirty = this.$refs.addService.isDirty();
      if (dirty) {
        this.$confirm("Some changes unsaved, discard?", "Unsaved", {
          confirmButtonText: "Yes",
          cancelButtonText: "No",
          type: "warning"
        })
          .then(() => {
            done();
          })
          .catch(() => {});
      } else {
        done();
      }
    },
    handleSelectionChange(val) {
      this.selectTable = val;
    },
    // eslint-disable-next-line no-unused-vars
    tableRowClassName({ row, rowIndex }) {
      if (rowIndex == this.tableErrorIndex - 1) {
        return "error";
      } else {
        return "";
      }
    },
    closeError() {
      this.$store.commit("setTableError", -1);
    },
    loadGroup() {
      this.itemIndex = "";
      var data = ipcRenderer.sendSync("readGroup");
      var map = new Map(JSON.parse(data));
      this.group = {};
      for (let [key, value] of map) {
        this.group[key] = JSON.parse(value);
      }
      for (var i in pregroup) {
        this.group[pregroup[i][0]] = JSON.parse(pregroup[i][1]);
      }
    },
    showNewCode(val) {
      this.codeIndex = val - 1;
    },
    showCode(item) {
      this.codeItem = item;
      this.jsFn = [];
      this.codeIndex = 0;
      if (item.type == "uds") {
        this.codeNum = 1;
        this.jsFn[0] = item.func;
      } else {
        for (var i in item.subtable) {
          this.jsFn[i] = item.subtable[i].func;
        }
        this.codeNum = item.subtable.length;
      }

      this.cd = true;
    },
    deleteMulti() {
      for (var i in this.selectTable) {
        let index = this.doipTable.indexOf(this.selectTable[i]);
        if (index !== -1) {
          this.deleteService(index);
        }
      }
    },
    deleteService(index) {
      if (this.mode === "doip") {
        this.$store.commit("doipTableDelete", index);
      } else if (this.mode === "can") {
        this.$store.commit("canTableDelete", index);
      } else if (this.mode === "lp") {
        this.$store.commit("lpTableDelete", index);
      } else {
        return;
      }
    },
    editItem(val) {
      var item=JSON.parse(JSON.stringify(val))
      if (this.mode === "doip") {
        this.$store.commit("doipItemChange", {
          index: this.editIndex,
          data: item
        });
      } else if (this.mode === "can") {
        this.$store.commit("canItemChange", {
          index: this.editIndex,
          data: item
        });
      } else if (this.mode === "lp") {
        this.$store.commit("lpItemChange", {
          index: this.editIndex,
          data: item
        });
      }
      this.cd1 = false;
    },
    editService(index, val) {
      this.editIndex = index;
      if (val.type == "uds") {
        for (let i in this.config) {
          if (this.config[i].value == val.service.value) {
            this.cd1 = true;
            this.refresh = false;
            this.service.type = "uds";
            this.service.cfg = this.config[i];
            this.service.val = JSON.parse(JSON.stringify(val));
            this.$nextTick(() => {
              this.refresh = true;
            });
            break;
          }
        }
      } else {
        this.loadGroup();
        for (let i in this.group) {
          if (i == val.service.name) {
            this.refresh = false;
            this.cd1 = true;
            this.service.type = "group";
            this.service.cfg = this.group[i];
            this.service.val = JSON.parse(JSON.stringify(val));
            this.$nextTick(() => {
              this.refresh = true;
            });
            break;
          }
        }
      }
    }
  },
  computed: {
    tableErrorIndex: function() {
      return this.$store.state.tableErrorIndex;
    },
    doipTable: function() {
      if (this.mode === "doip") {
        return this.$store.state.doipTable;
      } else if (this.mode === "can") {
        return this.$store.state.canTable;
      } else if (this.mode === "lp") {
        return this.$store.state.lpTable;
      } else {
        return [];
      }
    },
    running: function() {
      return this.$store.state.running;
    }
  }
};
</script>
<style>
.table_error {
  position: absolute;
  padding: 0px;
  margin: 0px;
  top: 0px;
  right: 0px;
}
.el-table .error {
  background: red;
}
.table_error i {
  z-index: 2;
  position: absolute;
  top: 0px;
  right: 0px;
  font-size: 25px;
  color: #e6a23c;
}
.name {
  font-size: 14px;
  font-weight: bolder;
  margin: 5px;
}
</style>
