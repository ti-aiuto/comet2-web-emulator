<template>
  <v-container>
    <v-btn type="primary" @click="executeNext">次の命令を実行</v-btn>

    <v-simple-table dense="dense">
      <tbody>
        <tr v-for="row of source">
          <td>{{ row[0] }}</td>
          <td>{{ row[1] }}</td>
          <td>{{ row[2] }}</td>
        </tr>
      </tbody>
    </v-simple-table>

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

    <v-alert
      v-for="(logMessage, index) of logMessages"
      :key="index"
      type="info"
      dense="dense"
    >
      {{ logMessage }}
    </v-alert>
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
        return window.prompt("結果を入力") || "";
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
  created() {
    this.loadAndcompile();
  },
  methods: {
    init() {
      this.memory = new Memory();
      this.register = new Register();
      this.machine = new Machine(this.memory, this.register, this.io);
      this.controller = this.machine.executeInteractive(0);
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
      this.logMessages.unshift(message);
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
        this.hasNext = await this.controller.executeNext();
        this.log("一命令実行しました。");
        this.generateMemoryDebugInfo();
      } catch (e) {
        this.hasNext = false;
        this.log("実行に失敗しました。");
        this.log(e);
      }
    },
  },
});
</script>
