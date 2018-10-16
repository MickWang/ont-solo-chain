<template>
  <div id="wrapper">
    <img id="logo" src="~@/assets/logo.png" alt="electron-vue">
    <main>
      <div>
        <a-button @click="chmodOntology">授权节点</a-button>
        <a-button @click="showPrikey">写入私钥文件</a-button>
        <a-button @click="startNode">启动节点</a-button>
        <a-button @click="stopNode">结束节点</a-button>
        <a-button @click="withdrawONG">提取ONG</a-button>
        <a-button @click="syncBlock">同步区块</a-button>        
      </div>
      <div style="margin-top:20px;">
        <a-button>
          <router-link class="link-tab" to="/transactions">交易列表</router-link>
        </a-button>
        <a-button>
          <router-link class="link-tab" to="/blocks">区块列表</router-link>
        </a-button>
        <a-button>
          <router-link class="link-tab" to="/events">Event列表</router-link>
        </a-button>
        <a-button>
          <router-link class="link-tab" to="/scs">合约交易列表</router-link>
        </a-button>
        
      </div>
    </main>
  
    <common-modal modalId="modal1" @modalOk="sysPassModalOk">
      <p>Input admin's password:</p>
      <a-input v-model="sysPass"></a-input>
    </common-modal>
  </div>
</template>

<script>
  import * as DB from '../../core/dbService.js'
  import CommonModal from './common/CommonModal' 
  import electron from 'electron'
  import { readFileSync, writeFile } from 'fs';
  import {Wallet, Crypto, OntAssetTxBuilder, RestClient} from 'ontology-ts-sdk'
  import {delay, getTxtype} from '../../core/util.js'
  var os = require('os').platform()
  console.log(os)
  var execFile = require('child_process').execFile;
  var exec = require('child_process').exec;


  var nodeProcess;
// const app = electron.remote.app;
// const userData = app.getAppPath()
// console.log('appPath: ' + userData)
  const path = require('path')
  const sudo = require('sudo-prompt')
  const options = {
    name: 'Solo Chain',
    // icns: '../../../build/icon.ions'
  }
  const url = 'http://127.0.0.1:20334'
  const rest = new RestClient(url)
  export default {
    name: 'landing-page',
    components:{
      CommonModal
    },
    mounted(){
      localStorage.removeItem('Node_PID')
    },
    data() {
      const node_accunts = JSON.parse(localStorage.getItem('node_accunts')) || []
      const hasChmod = localStorage.getItem('hasChmod') || false;
      const ontologyPath = path.join(__static, '/ontology-darwin-amd64')
      return {
        node_accunts,
        passModal:false,
        ontologyPath,
        hasChmod,
        sysPass: ''
      }
    },
    methods: {
      open (link) {
        this.$electron.shell.openExternal(link)
      },
      chmodOntology() {
        // this.$store.commit('SHOW_COMMON_MODAL')
        if(this.hasChmod === 'true') {
          this.$message.success('已授权。')
          return;
        }
        const command = 'chmod +x ' + this.ontologyPath
        sudo.exec(command, options, (error, stdout, stderr) => {
          if(error) throw error
          console.log('stdout: ' + stdout)
          localStorage.setItem('hasChmod', true);
          this.$message.success('授权成功。')
        })

      },
      startNode() {
        let command = '';
        if(os === 'darwin') {
          command = './ontology-darwin-amd64 --rest --ws --localrpc --gaslimit 20000 --gasprice 0 --testmode'
          nodeProcess = exec(command, {
            cwd: __static
          })
        } else if (os === 'linux') {
          command = './ontology-linux-amd64 --rest --ws --localrpc --gaslimit 20000 --gasprice 0 --testmode'
          nodeProcess = exec(command, {
            cwd: __static
          })
        } else if (os === 'win32') {
          command = 'ontology-windows-amd64.exe'
          nodeProcess = execFile(commit, ['--rest --ws --localrpc --gaslimit 20000 --gasprice 0 --testmode'],{
            cwd: __static
          })

        }
        
        nodeProcess.stdout.on('data', (data) => {
          console.log(data.toString())
        })
        nodeProcess.stdin.write('1\n');
        nodeProcess.stdin.end();

        setTimeout(()=>{
          this.syncBlock()
        }, 1000)

      // transfer to itself
      setTimeout(()=>{
        const command1 = 'cd ' + __static + ' && ./ontology-darwin-amd64 asset transfer --from 1 --to 1 --amount 1'
        const process1 = exec(command1,(error, stdout, stderr) => {
          if(error) throw error;
          console.log(stdout)
        })
        process1.stdin.write('1\n');
      }, 500)

        console.log(nodeProcess)
        localStorage.setItem('Node_PID', nodeProcess.pid)
      },
      stopNode() {
        if(localStorage.getItem('Node_PID')) {
          process.kill(parseInt(localStorage.getItem('Node_PID')))
        }
      },
      showPrikey(){
        const password = '1' // default
        const walletPath = __static + '/wallet.dat'
        const walletString = readFileSync(walletPath).toString();
        const wallet = Wallet.parseJson(walletString)
        const params = {
          cost: wallet.scrypt.n,
          blockSize: wallet.scrypt.r,
          parallel: wallet.scrypt.p,
          size: wallet.scrypt.dkLen
        }
        const privateKeys = wallet.accounts.length>0 && wallet.accounts.map((a) => {
          const enc = a.encryptedKey;
          const pri = enc.decrypt(password, a.address, a.salt, params);
          return {
            address: a.address.toBase58(),
            privateKey: pri.key
          }
        })
        writeFile(__static + '/privateKey.json', JSON.stringify(privateKeys), (err)=>{
          if(err) throw err;
          console.log('Complete privateKeys.json')
        });
      },
      sysPassModalOk(modalId) {
        console.log(modalId)
        this.$store.commit('HIDE_COMMON_MODAL')

      },
      withdrawONG() {
        // need to wait tranfer finish
        const command2 = 'cd ' + __static + ' && ./ontology-darwin-amd64 asset withdrawong 1'
          const process2 = exec(command2 ,(error, stdout, stderr) => {
            if(error) throw error;
            console.log(stdout)
          })
          process2.stdin.write('1\n')
      },
      async syncBlock() {
        let currentHeight = localStorage.getItem('Current_Height') || 0;
        currentHeight = parseInt(currentHeight)
        const height = (await rest.getBlockHeight()).Result
        if(currentHeight < height) {
          for(let i=currentHeight+1; i<=height; i++) {
            const block = (await rest.getBlockJson(i)).Result
            const event = (await rest.getSmartCodeEvent(i)).Result
            const blockData = {
              height: i,
              hash: block['Hash'],
              txNum: block.Transactions.length,
              json: block
            }
            DB.dbInsert(DB.blockDB, blockData).then(()=>{}, (err)=>{console.log(err)})
            if(event) {
              const eventData = {
                height: i,
                json: event 
              }
            DB.dbInsert(DB.eventDB, eventData).then(()=>{}, (err)=>{console.log(err)})
            }
            if(block && block.Transactions && block.Transactions.length > 0) {
              console.log(block.Transactions)
              for(const tx of block.Transactions) {
                const type = getTxtype(tx["TxType"])
                if(type === 'Deploy') {
                  const scData = {
                    height:i,
                    hash: tx['Hash'],
                    json:tx
                  }
                  DB.dbInsert(DB.scDB, scData).then(()=>{}, (err)=>{console.log(err)})
                }
                const txData = {
                  type: type,
                  hash: tx["Hash"],
                  height: i,
                  json: tx
                }
                DB.dbInsert(DB.txDB, txData).then(()=>{}, (err)=>{console.log(err)})
              }
            }
           
          }
          localStorage.setItem('Current_Height', height)
        }
        // const block = (await rest.getBlockJson(height)).Result
        // // alert(JSON.stringify(block))

        // const event = (await rest.getSmartCodeEvent(height)).Result
        // console.log(JSON.stringify(event))

      }
      
    }
  }
</script>

<style>
  @import url('https://fonts.googleapis.com/css?family=Source+Sans+Pro');

  * {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
  }

  body { font-family: 'Source Sans Pro', sans-serif; }

  #wrapper {
    background:
      radial-gradient(
        ellipse at top left,
        rgba(255, 255, 255, 1) 40%,
        rgba(229, 229, 229, .9) 100%
      );
    height: 100vh;
    padding: 60px 80px;
    width: 100vw;
  }

  #logo {
    height: auto;
    margin-bottom: 20px;
    width: 420px;
  }

  main {
    display: flex;
    flex-direction: column;
    justify-content: space-between;
  }


  .left-side {
    display: flex;
    flex-direction: column;
  }

  .welcome {
    color: #555;
    font-size: 23px;
    margin-bottom: 10px;
  }

  .title {
    color: #2c3e50;
    font-size: 20px;
    font-weight: bold;
    margin-bottom: 6px;
  }

  .title.alt {
    font-size: 18px;
    margin-bottom: 10px;
  }

  .doc p {
    color: black;
    margin-bottom: 10px;
  }

  .doc button {
    font-size: .8em;
    cursor: pointer;
    outline: none;
    padding: 0.75em 2em;
    border-radius: 2em;
    display: inline-block;
    color: #fff;
    background-color: #4fc08d;
    transition: all 0.15s ease;
    box-sizing: border-box;
    border: 1px solid #4fc08d;
  }

  .doc button.alt {
    color: #42b983;
    background-color: transparent;
  }
  .link-tab {
    margin: 10px;
  }
</style>
