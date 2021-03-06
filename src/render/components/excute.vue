
<template>
  <div>
    <div>
      <el-dialog title="Address" :visible.sync="showAddr" center width="30%">
        <el-select v-model="addrIndex" placeholder="Address" style="width:100%" size="mini">
          <el-option v-for="(item,key) in addrTable" :key="key" :label="item.name" :value="key">
            <span style="float: left">{{ item.name }}</span>
            <span
              style="float: right; color: #8492a6; font-size: 13px"
            >SA:{{ item.SA}},TA:{{item.TA}}</span>
          </el-option>
        </el-select>
        <span slot="footer">
          <el-button icon="el-icon-caret-right" @click="readRun" size="mini" type="primary">Start</el-button>
        </span>
      </el-dialog>
    </div>
    <el-row style="margin-top:10px;">
      <el-col :span="22" :offset="1" style="text-align:right">
        Log Level:
        <el-select
          v-model="logLevel"
          placeholder="Log Level"
          size="small"
          style="width:100px"
          @change="logLevelChange"
        >
          <el-option value="silly" />
          <el-option value="debug" />
          <el-option value="verbose" />
          <el-option value="info" />
          <el-option value="warn" />
          <el-option value="error" />
          <el-option value="false" />
        </el-select>Suppress Delay:
        <el-input
          v-model="sDelay"
          size="small"
          placeholder="Delay"
          style="width:80px"
          maxlength="6"
        ></el-input>ms,
        Timeout:
        <el-input
          v-model="udsTimeout"
          size="small"
          placeholder="Timeout"
          style="width:80px"
          maxlength="6"
          :disabled="mode==='lp'"
        ></el-input>ms,
        Cycle:
        <el-input-number
          v-model.number="cycle"
          controls-position="right"
          :min="1"
          size="small"
          style="width:100px"
        ></el-input-number>
        <!-- <el-button @click="run" size="small" type="success" :disabled="!connected||running">开始</el-button> -->
      </el-col>
    </el-row>
    <el-row style="margin-top:10px;">
      <el-col :span="22" :offset="1" style="text-align:right">
        <el-button @click="run" size="small" type="success" :disabled="!connected||running">Start</el-button>
      </el-col>
    </el-row>
    <div>
      <pre id="clog"></pre>
    </div>
    <!-- <el-input readonly type="textarea" :rows="5" placeholder="LOG" v-model="logText" id="log"></el-input> -->
  </div>
</template>
<script>
/* eslint-disable no-unused-vars */
const { ipcRenderer } = require("electron");
const log = require("electron-log");
const util = require("util");
var AU = require("ansi_up");
var ansi_up = new AU.default();
export default {
  data() {
    return {
      cycle: 1,
      interCycle: 1,
      startTime: "",
      udsTimeout: "2000",
      sDelay: "100",
      logText: "",
      showAddr: false,
      addrIndex: "",
      logLevel: "info",
    };
  },
  mounted() {
    this.logLevel = this.$store.state.logLevel;
    ipcRenderer.on("udsEnd", (event, val) => {
      this.success(val);
      if (this.interCycle > 1) {
        this.interCycle--;
        this.logText += `\x1B[1;;33mStart again:${this.interCycle}\x1B[0m\n`;
        this.readRun();
      }
    });
    ipcRenderer.on(log.transports.ipc.eventId, (event, message) => {
      const text = util.format.apply(util, message.data);
      let msg;
      let usedTime = Date.parse(message.date) - this.startTime;
      if (usedTime < 0) {
        usedTime = 0;
      }
      if (message.level == "info") {
        msg = `\x1B[1;;32m[${usedTime}ms] ${text}\x1B[0m\n`;
      } else if (message.level == "error") {
        msg = `\x1B[1;;31m[${usedTime}ms] ${text}\x1B[0m\n`;
      } else if (message.level == "warn") {
        msg = `\x1B[1;;33m[${usedTime}ms] ${text}\x1B[0m\n`;
      } else {
        msg = `[${usedTime}ms] ${text}\r\n`;
      }
      this.logText += msg;
    });
    ipcRenderer.on("udsError", (event, val) => {
      this.failed(val.msg);
      this.$store.commit("setTableError", val.index);
    });
  },
  destroyed() {
    ipcRenderer.removeAllListeners("udsEnd");
    ipcRenderer.removeAllListeners(log.transports.ipc.eventId);
    ipcRenderer.removeAllListeners("udsError");
  },
  props: {
    mode: {
      type: String,
      default: function () {
        return "can";
      },
    },
  },
  watch: {
    logText: function (val) {
      var cdiv = document.getElementById("clog");
      var html = ansi_up.ansi_to_html(val);
      cdiv.innerHTML = html;
      this.$nextTick(() => {
        let container = document.getElementById("clog");
        container.scrollTop = container.scrollHeight;
      });
    },
  },
  computed: {
    connected: function () {
      if (this.mode === "can") {
        return this.$store.state.canConnect;
      } else if (this.mode === "doip") {
        return this.$store.state.doipConnect;
      } else if (this.mode === "lp") {
        return true;
      } else {
        return false;
      }
    },
    udsTable: function () {
      if (this.mode === "can") {
        return this.$store.state.canTable;
      } else if (this.mode === "doip") {
        return this.$store.state.doipTable;
      } else if (this.mode === "lp") {
        return this.$store.state.lpTable;
      } else {
        return [];
      }
    },
    running: function () {
      return this.$store.state.running;
    },
    addrTable: function () {
      if (this.mode === "doip") {
        return this.$store.state.doipAddrTable;
      } else if (this.mode === "can") {
        return this.$store.state.canAddrTable;
      } else if (this.mode === "lp") {
        return this.$store.state.lpAddrTable;
      } else {
        return [];
      }
    },
  },
  methods: {
    logLevelChange() {
      ipcRenderer.send("logLevel", this.logLevel);
      this.$store.commit("logLevel", this.logLevel);
    },
    failed(msg) {
      this.$store.commit("runChange", false);
      this.$notify.error({
        title: "Error",
        message: msg,
        duration: 0,
      });
    },
    success(msg) {
      this.$store.commit("runChange", false);
      this.$notify({
        title: "Success",
        message: msg,
        type: "success",
      });
    },
    readRun() {
      this.startTime = new Date().getTime();
      var table = JSON.parse(JSON.stringify(this.udsTable));
      if (this.addrTable[this.addrIndex]) {
        this.showAddr = false;
      } else {
        this.$notify.error({
          title: "Error",
          message: "Please choose correct address.",
        });
        return;
      }
      this.$store.commit("runChange", true);
      this.$store.commit("setTableError", -1);
     

      ipcRenderer.send(this.mode + "udsExcute", {
        addr: this.addrTable[this.addrIndex],
        udsTable: table,
        timeout: parseInt(this.udsTimeout, 10),
        sDelay: parseInt(this.sDelay, 10),
      });
    },
    run() {
      this.logText = "";
      this.showAddr = true;
      this.interCycle = this.cycle;
      // console.log(this.udsTable)
    },
  },
};
</script>
<style>
#clog {
  height: 150px;
  padding: 10px;
  border-radius: 5px;
  border-style: solid;
  border-width: 2px;
  border-color: gray;
  font-size: 15px;
  /* font-family: "Helvetica Neue",Helvetica,"PingFang SC","Hiragino Sans GB","Microsoft YaHei","微软雅黑",Arial,sans-serif; */
  overflow: auto;
}
</style>
