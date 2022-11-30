<template>
  <a-card @dragover.prevent @dragenter.prevent @drop="onDrop" @dragstart="dragStart" draggable hoverable v-if="selectedCards.includes(cards_data.title)">
    <div v-if="cards_data.extra" style="width: 150px;float:right">
      <a class="progress-title">{{cards_data.extra[0].title.toUpperCase()}}: ({{cards_data.extra[0].value}}%)</a>
      <a-progress size="small" status="normal" :format="percent => ``" :percent="cards_data.extra[0].value">more</a-progress>
    </div>
    <br/>
    <h3><strong>{{cards_data.title.toUpperCase()}} </strong></h3>
    <div v-for="items in cards_data.data" :key="items.id">
      <hr/>
      <h4><strong>{{items.title.toUpperCase()}}</strong></h4>
      <div v-if="items.type==='progress'">
        <div>
          <div style="float:left" v-for="progress in items.value" :key="progress.id">
            <div style="width: 150px">
              <a class="progress-title">{{progress.title}}: ({{progress.value}}%)</a>
              <a-progress size="small" :percent="progress.value" :format="percent => ``"></a-progress>
            </div>
          </div>
          <br/><br/>
        </div>
      </div>
      <h4 class="cut" v-else>{{items.value}}</h4>
    </div>
  </a-card>
</template>
<script>
export default ({
  props: {
    data: Object
  },
  data () {
    return {
      cards_data: this.data,
      selectedCards: []
    }
  },
  timers: {
    update: { time: 500, autostart: true, immediate: true, repeat: true }
  },
  methods: {
    onDrop (event) {
      const name = event.dataTransfer.getData('name')
      this.$emit('changePositions', name, this.cards_data.title)
    },
    dragStart (event) {
      event.dataTransfer.setData('name', this.cards_data.title)
    },
    async update () {
      await this.$uci.load('overview')
      const sid = await this.$uci.sections('overview', 'cards')
      this.selectedCards = await sid[0].visibleCards
    }
  },
  async created () {
    await this.$uci.load('overview')
    const sid = await this.$uci.sections('overview', 'cards')
    this.selectedCards = await sid[0].visibleCards
  }
})
</script>
<style>
.progress {
display:inline;
}
.progress-title{
    font-size:12px
}
.cut {
  text-overflow: ellipsis;
  overflow: hidden;
  width: 310px;
  height:2em;
  white-space: nowrap;
}
</style>
