<template>

<div class="modal" :class="{'is-active': showModal}">
  <div class="modal-background"></div>
  <div class="modal-card" style="width:500px">
    <header class="modal-card-head">
      <p class="modal-card-title is-size-4 has-text-link has-text-weight-semibold">{{ $t("msg.gnode.title") }}</p>
      <button class="delete" aria-label="close" @click="closeModal"></button>

    </header>
    <section class="modal-card-body" style="height:540px;background-color: whitesmoke;">
      
       <div class="container">
        <div class="columns">
            <div class="column is-12">
                <div class="tabs">
                    <ul>
                        <li :class="[ tab === 'status' ? 'is-active' : '']"><a @click="tab='status'">{{ $t("msg.gnode.tabStatus") }}</a></li> 
                        <li :class="[ tab === 'peers' ? 'is-active' : '']"><a @click="tab='peers'">{{ $t("msg.gnode.tabPeers") }}</a></li>
                        <li :class="[ tab === 'log' ? 'is-active' : '']"><a @click="tab='log'">{{ $t("msg.gnode.tabLog") }}</a></li>
                        <li :class="[ tab === 'config' ? 'is-active' : '']"><a @click="tab='config'">{{ $t("msg.gnode.tabConfig") }}</a></li>
                    </ul>
                </div>
                <div class="tab-is-link">
                    <div v-if="tab ==='status'">
                      <p>{{ $t("msg.gnode.status") }}&nbsp;: 
                        <span class="has-text-centered has-text-weight-semibold">{{getStatusDisplay()}}</span>
                        <span v-if="status==='syncing'" class="animated flash" style="animation-iteration-count:infinite;animation-duration: 3s">
                        ......</span>
                      </p>
                      
                      <div v-if="status!='toStart'">
                        <p>{{ $t("msg.gnode.localRemoteHeight") }}&nbsp;: 
                          <span class="has-text-centered has-text-weight-semibold">{{localHeight}}/({{remoteHeight}})</span>
                        </p>
                        <br/>
                        <p>{{ $t("msg.gnode.nodeVersion") }}&nbsp;: 
                          <span class="has-text-centered has-text-weight-semibold">{{userAgent}}</span>
                        </p>
                        <p>{{ $t("msg.gnode.protocolVersion") }}&nbsp;: 
                          <span class="has-text-centered has-text-weight-semibold">{{protocolVersion}}</span>
                        </p>
                        <br/>
                        <p>{{ $t("msg.gnode.connectedCount") }}&nbsp;: 
                          <span class="has-text-centered has-text-weight-semibold">{{peers.length}}</span>
                        </p>
                        <br/>
                        <p>{{ $t("msg.gnode.location") }}&nbsp;: </p>
                        <p><span class="has-text-centered has-text-weight-semibold">{{chainDataPath}}</span></p>
                        <br/>
                        <p v-if="chainDataSize != 0">{{ $t("msg.gnode.size") }} &nbsp;: 
                          <span class="has-text-centered has-text-weight-semibold" style="white-space: nowrap">{{chainDataSize}}</span>
                        </p>

                        <br/><br/>
                        
                        <div class="field is-grouped">
                          <div class="control">
                            <button class="button is-link is-small" @click="restart">{{ $t("msg.gnode.restart") }}</button>
                          </div>
                          <div class="control">
                            <button class="button is-small is-text is-multiline" v-if="removed" disabled>{{ $t("msg.gnode.remove") }}</button>
                            <button class="button is-small is-text is-multiline" @click="remove" v-else>{{ $t("msg.gnode.remove") }}</button>
                          </div>
                        </div>

                      </div>
                      <div class="control" v-else>
                        <br/>
                        <button class="button is-warning is-multiline" v-if="removed" disabled>{{ $t("msg.gnode.remove") }}</button>
                        <button class="button is-warning is-multiline" @click="remove" v-else>{{ $t("msg.gnode.remove") }}</button>
                      </div>
                      
                    </div>
                    <div v-if="tab ==='peers'">
                      <p class="tag is-warning">{{ $t("msg.gnode.connectedCount") }}: {{peers.length}}</p>
                      <br/>
                      <br/>

                      <table v-if="peers.length > 0" class="table is-striped is-narrow is-hoverable" style="font-size:0.8rem;background-color: #f6f9fe;color: #22509a;">
                        <thead>
                          <th>No.</th>
                          <th>IP</th>
                          <th>{{ $t("msg.node") }}</th>
                          <th>{{ $t("msg.gnode.height") }}</th>
                        </thead>
                        <tbody>
                          <tr v-for="(peer, index) in peers" :key="peer.id">
                            <td>{{index+1}}</td>
                            <td>{{peer.addr}}</td>
                            <td>{{peer.user_agent}}</td>
                            <td>{{peer.height}}</td>
                          </tr>
                        </tbody>
                      </table>
                    </div>
                    <div v-if="tab ==='log'">
                        <p class="is-size-7" v-for="log in nodeLog" :key="log.id">{{ log }}</p>
                    </div>
                    <div v-if="tab ==='config'">
                      <GnodeConfig v-on:close="closeModal"></GnodeConfig>
                    </div>
                    
                </div>
              </div>
            </div>
        </div>
    </section>
  </div>
</div>

</template>
<script>
import { messageBus } from '@/messagebus'
import {gnodeOption, grinNodeLog, chainDataPath} from '../../shared/config'

import GnodeConfig from '@/components/GnodeConfig'

import { remote } from 'electron';
import { setTimeout } from 'timers';
const Tail = require('tail').Tail;
const getSize = require('get-folder-size');

export default {
  name: "gnode",
  props: {
    showModal: {
      type: Boolean,
      default: false
    }
  },
  components: {
      GnodeConfig
  },
  data() {
    return {
      tab: 'status', //status, peers, log, config
      status: 'toStart',//'toStart','syncing','running'
      localHeight: -1,
      remoteHeight: -1,
      userAgent: '',
      protocolVersion: '',
      peers: [],
      nodeLog: [],
      chainDataPath: chainDataPath,
      chainDataSize: 0,
      removed: false
    }
  },
  watch: {
    status:function(newVal, oldVal){
      this.$dbService.setLocalGnodeStatus(newVal)
    }
  },
  created(){
    messageBus.$on('gnodeStarting',()=>{
      this.$log.debug('Got event gnodeStarting')
      setTimeout(()=>this.checkStarted(), 5000)
    })
    setTimeout(()=>{
    if(gnodeOption.useLocalGnode){
      this.$gnodeService.getStatus().then().catch((err)=>{
        this.$log.debug('Try to start gnode after 45s')
        this.$gnodeService.startGnode()
      })
    }}, 45*1000)
  },
  mounted() {
    this.checkStarted()
    this.getChainDataSize()
    setTimeout(()=>{
      if(this.status == 'toStart'){
        this.checkStarted()
      }
    }, 10*1000)
  },
  methods: {
    checkStarted(){
      this.$gnodeService.getStatus().then((res)=>{
        this.checkStatus()
        this.getPeers()
        this.startTailLog(this.nodeLog)
        this.autoCheck(10 * 1000)
      }).catch((error)=>{
        this.status = 'toStart'
        this.$dbService.setLocalGnodeStatus('toStart')
      })
    },

    checkStatus(){
      let checkStatusAsync = async function(){
        try{
          let res = await this.$gnodeService.getStatus()
          let data = res.data.result.Ok
          this.localHeight = parseInt(data.tip.height)
          this.userAgent = data.user_agent
          this.protocolVersion = data.protocol_version

          let res2 = await this.$remoteGnodeService.getStatus()
          let data2 = res2.data.result.Ok
          this.remoteHeight = parseInt(data2.tip.height)
          //console.log(`remote ${thiis.remoteHeight}; local ${this.localHeight}`)

        if( this.localHeight + 60 > this.remoteHeight){
          this.status = 'running'
        }else{
          this.status = 'syncing'
        }}catch(error){
          console.log(error)
          this.status = 'toStart'
        }
      }
      checkStatusAsync.call(this)
    },

    getPeers(){
      this.$gnodeService.getConnectedPeers().then(
        (res)=>{
         this.peers = res.data.result.Ok
      }).catch(
        (error)=>{
          this.$log.debug('Failed to get peers: ' + error)
          this.peers = []
        }
      )
    },

    getChainDataSize(){
      getSize(chainDataPath, (err, size) => {
        if (err) { this.$log.error('Error when get size of chain data dir: ' + err)}
        this.chainDataSize = (size / 1024 / 1024).toFixed(2) + ' MB'
      })
    },

    startTailLog(log){
      let tail = new Tail(grinNodeLog, {'useWatchFile':true})
      //console.log(grinNodeLog)
      tail.on("line", function(data) {
        //console.log(data)
        if(log.length >= 30){log.shift()}
        log.push(data)
      });

      tail.on("error", function(error) {
        log.push('Error: ' + error)
      });
    },

    getStatusDisplay(){
      let s = {
        'running': this.$t('msg.gnode.statusRunning'),
        'syncing': this.$t('msg.gnode.statusSyncing'),
        'toStart': this.$t('msg.gnode.statusToStart')
      }
      return s[this.status]
    },

    autoCheck(interval){
        setInterval(()=>{
         this.checkStatus()
         this.getPeers()
         this.getChainDataSize()
        }, interval)
    },
    restart(){
      this.$gnodeService.restartGnode()
      this.status = 'toStart'
      this.nodeLog = []
    },
    closeModal() {
      messageBus.$emit('close', 'windowGnode');
    },
    remove(){
      let result = this.$electron.remote.dialog.showMessageBox(null, {
        type: 'question',
        buttons: ['Yes', 'No'],
        title: 'Question',
        message: this.$t('msg.gnode.removeConfirm')
      })

      if(result===0){
        this.removed = true
        this.$gnodeService.stopGnode2()
        setTimeout(() => {
          this.$walletService.removeChainData()
          this.$log.debug('ChainData removed.')
          this.restart() 
        }, 4000);
      }
    }
  }
}
</script>
<style scoped>
.center{
  display: flex;
  justify-content: center;
  align-items: center;
}
li.is-active{
  background-color: #3273dc;
  color: #fff;
  border-radius: 4px 4px 0 0;
}

.tabs li.is-active a{
  color: #fff;
}

.tab-is-link{
  background-color: #f6f9fe;
  border-radius: 4px;
  color: #22509a;
  padding: 1.5em 1.8em;
  margin-top: -1.5rem
}

.is-size-7{
  font-size: 0.95rem
}

.button.is-multiline {
  white-space: normal;
  height: auto;
  max-width: 250px;
  flex-direction: column;
}

</style>
