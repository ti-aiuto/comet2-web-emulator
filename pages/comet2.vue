<template>
  <v-container>
    <p>
      ソース<br />
      https://github.com/ti-aiuto/comet2-emulator<br />
      https://github.com/ti-aiuto/comet2-web-emulator<br />
    </p>
    <v-btn @click="loadAndcompile">コンパイル実行</v-btn>
    <v-btn color="primary" @click="executeNext" :disabled="!hasNext"
      >次の命令を実行</v-btn
    >
    <br />
    <v-btn @click="loadSource(0)">最小公倍数を求めるやつ</v-btn>
    <v-btn @click="loadSource(1)">なんか課題できたやつ</v-btn>
    <v-btn @click="loadSource(2)">入出力の動作確認用</v-btn>

    <h3>ログ</h3>
    <div class="log-messages">
      <v-alert
        v-for="(logMessage, index) of logMessages"
        :key="index"
        type="info"
        dense="dense"
        outlined="outlined"
      >
        {{ logMessage }}
      </v-alert>
    </div>

    <v-row>
      <v-col cols="12" md="6">
        <h3>レジスタ</h3>
        <v-simple-table dense="dense">
          <tbody>
            <tr>
              <td>PC</td>
              <td>{{ register.getProgramCounter() }}</td>
            </tr>
            <tr>
              <td>OF</td>
              <td>{{ register.getOverflowFlag() }}</td>
            </tr>
            <tr>
              <td>SF</td>
              <td>{{ register.getSignFlag() }}</td>
            </tr>
            <tr>
              <td>ZF</td>
              <td>{{ register.getZeroFlag() }}</td>
            </tr>
            <tr>
              <td>GR0</td>
              <td>{{ register.getGRAt(0) }}</td>
            </tr>
            <tr>
              <td>GR1</td>
              <td>{{ register.getGRAt(1) }}</td>
            </tr>
            <tr>
              <td>GR2</td>
              <td>{{ register.getGRAt(2) }}</td>
            </tr>
            <tr>
              <td>GR3</td>
              <td>{{ register.getGRAt(3) }}</td>
            </tr>
            <tr>
              <td>GR4</td>
              <td>{{ register.getGRAt(4) }}</td>
            </tr>
            <tr>
              <td>GR5</td>
              <td>{{ register.getGRAt(5) }}</td>
            </tr>
            <tr>
              <td>GR6</td>
              <td>{{ register.getGRAt(6) }}</td>
            </tr>
            <tr>
              <td>GR7</td>
              <td>{{ register.getGRAt(7) }}</td>
            </tr>
          </tbody>
        </v-simple-table>
      </v-col>
      <v-col cols="12" md="6">
        <h3>メモリ</h3>
        <v-simple-table dense="dense">
          <tbody>
            <tr v-for="row of memoryDebugInfo">
              <td>{{ row[0] }}</td>
              <td>{{ row[1] }}</td>
              <td>{{ row[2] }}</td>
              <td>{{ row[3] }}</td>
            </tr>
          </tbody>
        </v-simple-table>
      </v-col>
    </v-row>

    <v-row>
      <v-col cols="6">
        <h3>ソース</h3>
        <v-simple-table dense="dense">
          <tbody>
            <tr v-for="row of source">
              <td>{{ row[0] }}</td>
              <td>{{ row[1] }}</td>
              <td>{{ row[2] }}</td>
            </tr>
          </tbody>
        </v-simple-table>
      </v-col>
    </v-row>
  </v-container>
</template>
<script lang="ts">
import Vue from "vue";
import { Memory } from "ts-comet2-emulator/dist/memory";
import { Register } from "ts-comet2-emulator/dist/register";
import { Compiler } from "ts-comet2-emulator/dist/compiler";
import { Machine } from "ts-comet2-emulator/dist/machine";
import {
  parseSource,
  toWordHex,
  memoryDebugInfo,
} from "ts-comet2-emulator/dist/utils";
import { IO } from "ts-comet2-emulator/dist/io";
import { castle2Examples } from "../lib/examples";

export default Vue.extend({
  data() {
    const io = new IO(
      async function() {
        return window.prompt("ascii文字を入力") || "";
      },
      async function(value: string) {
        alert(value);
      }
    );

    const memory = new Memory();
    const register = new Register();
    const machine = new Machine(memory, register, io);

    const source = castle2Examples[0];

    return {
      io,
      memory,
      register,
      machine,
      source,
      controller: machine.executeInteractive(0),
      logMessages: [] as string[],
      memoryDebugInfo: memoryDebugInfo(memory.dump(), {}, source),
      addrToSourceIndexMap: {},
      hasNext: false,
    };
  },
  created() {},
  methods: {
    init() {
      this.memory = new Memory();
      this.register = new Register();
      this.machine = new Machine(this.memory, this.register, this.io);
      this.controller = this.machine.executeInteractive(0);
    },
    loadSource(index: number) {
      this.source = castle2Examples[index];
    },
    loadAndcompile() {
      this.init();
      const firstAddress = 0;
      const labelMap = {};
      const compiler = new Compiler(
        this.memory,
        firstAddress,
        this.source,
        labelMap
      );
      try {
        compiler.compile();
        this.addrToSourceIndexMap = compiler.addrToSourceIndexMap();
        this.generateMemoryDebugInfo();
        this.log("コンパイル完了");
        this.hasNext = true;
      } catch (e) {
        this.log("コンパイル失敗");
        this.log(e);
      }
    },
    log(message: string) {
      this.logMessages.unshift(`${new Date().toISOString()} ${message}`);
    },
    generateMemoryDebugInfo() {
      this.memoryDebugInfo = memoryDebugInfo(
        this.memory.dump(),
        this.addrToSourceIndexMap,
        this.source
      );
    },
    async executeNext() {
      if (!this.hasNext) {
        return;
      }
      try {
        const result = await this.controller.executeNext();
        this.hasNext = result;
        this.log("一命令実行しました。");
        this.generateMemoryDebugInfo();
        if (!result) {
          this.log("実行完了");
        }
      } catch (e) {
        this.hasNext = false;
        this.log("実行に失敗しました。");
        this.log(e);
      }
    },
  },
});
</script>

<style scoped>
.log-messages {
  height: 100px;
  overflow: auto;
}
</style>
