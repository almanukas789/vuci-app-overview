<template>
  <div>
    <div v-if="this.selectedCards.length!==0 && !this.loading" class="cards">
      <overviewCard v-for="item in cards" :data="item" :key="item.title" @changePositions="changePositions" />
    </div>
    <div v-else style="display:grid;place-items:center">No selected cards</div>
      <drawer :cards="cards" @cardsSettings="cardsSettings"></drawer>
  </div>
</template>
<script>
import overviewCard from './components/card.vue'
import drawer from './components/drawerComponent.vue'
export default {
  components: {
    overviewCard,
    drawer
  },
  timers: {
    update: { time: 1000, autostart: true, immediate: true, repeat: true }
  },
  watch: {
    cards: {
      handler (newValue, oldValue) {
        this.cards = newValue
      },
      deep: true
    }
  },
  data: function () {
    return {
      cards: [],
      lastCPUTime: null,
      cpuPercentage: 0,
      selectedCards: [],
      position: [],
      loading: ''
    }
  },
  methods: {
    async changePositions (from, to) {
      const fromIndex = this.position.indexOf(from)
      const toIndex = this.position.indexOf(to)
      const element = this.position.splice(fromIndex, 1)[0]
      this.position.splice(toIndex, 0, element)
      var obj = []
      for (let i = 0; i < Object.keys(this.position).length; i++) {
        for (let j = 0; j < Object.keys(this.cards).length; j++) {
          if (this.cards[j].title.includes(this.position[i])) {
            obj[i] = this.cards[j]
          }
        }
      }
      this.cards = obj
      this.savePositions(this.position)
    },
    async  savePositions (self) {
      await this.$uci.load('overview-pos')
      const checkSid = await this.$uci.sections('overview-pos')
      if (Object.keys(checkSid).length === 0) {
        const sid = this.$uci.add('overview-pos', 'cards')
        this.$uci.set('overview-pos', sid, 'cardsPosition', self)
      } else {
        const existSid = this.$uci.sections('overview-pos', 'cards')
        this.$uci.set('overview-pos', existSid[0]['.name'], 'cardsPosition', self)
      }
      this.$uci.save()
      this.$uci.apply()
      this.$emit('cardsSettings', self)
    },
    async cardsSettings (cards) {
      this.selectedCards = await cards
    },
    async  update () {
      var test = await this.getSystemData()
      const updateSystemCard = await this.cards.find(result => result.title.toLowerCase() === 'system')
      updateSystemCard.extra[0].value = this.cpuPercentage
      updateSystemCard.data = test.data
    },
    async getSystemData () {
      const data = { title: 'System', data: [], extra: [] }
      const systemInfo = await this.$system.getInfo().then(r => { return r })
      data.data.push({ title: 'Router uptime', value: '%t'.format(systemInfo.uptime) })
      data.data.push({ title: 'Local device time', value: this.formatDate(systemInfo.localtime) })
      data.data.push({
        title: 'Memory usage',
        type: 'progress',
        value: [
          { title: 'RAM', value: Math.floor(((systemInfo.memory.total - systemInfo.memory.free) / systemInfo.memory.total) * 100) },
          { title: 'FLASH', value: Math.floor((systemInfo.disk.root.used / systemInfo.disk.root.total) * 100) }
        ]
      })
      data.data.push({ title: 'Firmware version', value: systemInfo.release.revision })
      this.cpuUsage()
      data.extra.push({ title: 'Cpu load', value: this.cpuPercentage })
      return data
    },
    async getInterfacesData () {
      await this.$network.load()
      const interfaces = await this.$network.getInterfaces()
      for (let i = 0; i < Object.keys(interfaces).length; i++) {
        const data = {
          title: interfaces[i].name.toUpperCase(),
          data: [{ title: 'Type', value: !interfaces[i].status.device ? 'Not Available' : interfaces[i].status.device },
            { title: 'Ip address', value: !interfaces[i].status['ipv4-address'] ? 'Not Available' : interfaces[i].status['ipv4-address'][0].address + '/' + interfaces[i].status['ipv4-address'][0].mask }]
        }
        this.cards.push(data)
      }
    },
    async getSystemEvents () {
      const events = await this.$log.get({ table: 'SYSTEM', limit: 5 })
      const data = { title: 'Recent system events', data: [] }
      for (let i = 0; i < 4; i++) {
        data.data.push({ title: this.formatDate(events.log[i].TIME), value: events.log[i].TEXT })
      }
      return data
    },
    async getNetworkEvents () {
      const events = await this.$log.get({ table: 'NETWORK', limit: 5 })
      const data = { title: 'Recent network events', data: [] }
      for (let i = 0; i < 4; i++) {
        data.data.push({ title: this.formatDate(events.log[i].TIME), value: events.log[i].TEXT })
      }
      return data
    },
    formatDate (item) {
      const d = new Date(item * 1000)
      const year = d.getFullYear().toString().padStart(2, 0)
      const month = (d.getMonth() + 1).toString().padStart(2, 0)
      const date = d.getDate().toString().padStart(2, 0)
      const hour = d.getHours()
      const min = d.getMinutes()
      const sec = d.getSeconds()
      return `${year}-${month}-${date} %02d:%02d:%02d`.format(hour, min, sec)
    },
    cpuUsage () {
      this.$rpc.call('system', 'cpu_time').then(times => {
        if (!this.lastCPUTime) {
          this.cpuPercentage = 0
          this.lastCPUTime = times
          return
        }

        const idle1 = this.lastCPUTime[3]
        const idle2 = times[3]

        let total1 = 0
        let total2 = 0

        this.lastCPUTime.forEach(t => {
          total1 += t
        })

        times.forEach(t => {
          total2 += t
        })

        this.cpuPercentage = Math.floor(((total2 - total1 - (idle2 - idle1)) / (total2 - total1)) * 100)
        this.lastCPUTime = times
      })
    },
    async cardPositions () {
      await this.$uci.load('overview-pos')
      const sid = await this.$uci.sections('overview-pos')
      if (sid.length === 0) {
        this.$spin(false)
        this.cards.forEach(r => {
          this.position.push(r.title)
        })
        console.log(this.position)
        console.log(this.cards)
        this.reorderCards()
        this.loading = false
        return
      }
      this.position = sid[0].cardsPosition
      if (this.cards.length > this.position.length) {
        for (let i = 0; i < this.cards.length; i++) {
          if (!this.position.includes(this.cards[i].title)) {
            this.position.push(this.cards[i].title)
          }
        }
      } else if (this.cards.length < this.position.length) {
        for (let i = 0; i < this.position.length; i++) {
          var contains = true
          for (let j = 0; j < this.cards.length; j++) {
            if (this.position[i] === this.cards[j].title) {
              contains = false
            }
          }
          if (contains) {
            this.position.splice(i, 1)
          }
        }
      }
      this.reorderCards()
      this.$spin(false)
      this.loading = false
    },
    async reorderCards () {
      var obj = []
      for (let i = 0; i < Object.keys(this.position).length; i++) {
        for (let j = 0; j < Object.keys(this.cards).length; j++) {
          if (await this.cards[j].title.includes(this.position[i])) {
            obj[i] = this.cards[j]
          }
        }
      }
      this.cards = obj
    }
  },
  async created () {
    this.loading = true
    this.$spin(true)
    this.cards.push(await this.getSystemData())
    this.cards.push(await this.getNetworkEvents())
    this.cards.push(await this.getSystemEvents())
    await this.getInterfacesData()
    await this.cardPositions()
  }
}
</script>
<style>
.cards {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(330px, 1fr));
  grid-auto-rows: 1fr;
  grid-gap: 15px;
}
</style>
