<template>
  <div class="monsters">

    <div class="d-flex flex-wrap align-center">
      <Breadcrumb class="mr-auto mb-4" />
      <div class="d-flex flex-wrap align-center">
        <v-btn :outlined="!$store.state.l5r" color="primary" class="mr-4 mb-4" depressed @click="$store.commit('setL5r', !$store.state.l5r)"><span class="orn">8</span> Règles cinq royaumes</v-btn>
        <v-btn color="primary" class="mr-4 mb-4" depressed link to="/creation-de-monstre-pnj/"><v-icon left>mdi-plus</v-icon> Créer un monstre/PNJ</v-btn>
        <MyMonstersButton />
      </div>
    </div>

    <h1>Bestiaire</h1>

    <div class="columns-toggles d-md-flex d-none align-center mb-2">
      <div><strong>Colonnes affichées :</strong></div>
      <v-checkbox
        class="mt-0 mr-4"
        v-model="showColumn.type"
        label="Type"
        hide-details
        @change="setShowColumn"
      ></v-checkbox>
      <v-checkbox
        class="mt-0 mr-4"
        v-model="showColumn.size"
        label="Taille"
        hide-details
        @change="setShowColumn"
      ></v-checkbox>
      <v-checkbox
        class="mt-0 mr-4"
        v-model="showColumn.subtype"
        label="Sous-type"
        hide-details
        @change="setShowColumn"
      ></v-checkbox>
      <v-checkbox
        class="mt-0 mr-4"
        v-model="showColumn.environments"
        label="Environnements"
        hide-details
        @change="setShowColumn"
      ></v-checkbox>
      <v-checkbox
        class="mt-0 mr-4"
        v-model="showColumn.dungeonTypes"
        label="Type de donjons"
        hide-details
        @change="setShowColumn"
      ></v-checkbox>
      <v-checkbox
        class="mt-0 mr-4"
        v-model="showColumn.encounter"
        label="Rencontre"
        hide-details
        @change="setShowColumn"
      ></v-checkbox>
    </div>

    <div class="active-filters mb-2">
      <div class="challengeRange-filter" v-if="Number(challengeRange[0]) >= 0 && Number(challengeRange[1]) <= challenges.length-1">
        <strong>Indice de dangerosité</strong> entre {{ challenges[challengeRange[0]].label }} et {{ challenges[challengeRange[1]].label }}
      </div>
      <div class="types-filter mb-1" v-if="selectedTypes.length > 0">
        <strong>Types</strong> : <v-chip class="mr-1" v-for="(type, idx) in selectedTypes">{{ type }}</v-chip>
      </div>
      <div class="sizes-filter mb-1" v-if="selectedSizes.length > 0">
        <strong>Tailles</strong> : <v-chip class="mr-1" v-for="(size, idx) in selectedSizes">{{ size }}</v-chip>
      </div>
      <div class="environments-filter mb-1" v-if="selectedEnvironments.length > 0">
        <strong>Environnements</strong> : <v-chip class="mr-1" v-for="(env, idx) in selectedEnvironments">{{ env }}</v-chip>
      </div>
      <div class="dungeon-types-filter mb-1" v-if="selectedDungeonTypes.length > 0">
        <strong>Types de donjons</strong> : <v-chip class="mr-1" v-for="(dType, idx) in selectedDungeonTypes">{{ dType }}</v-chip>
      </div>
    </div>

    <v-data-table
      class="data-table"
      :headers="headers"
      :items="monsters"
      item-key="key"
      :sort-by.sync="sortBy"
      :sort-desc.sync="sortDesc"
      must-sort
      :search="search"
      :items-per-page.sync="itemsPerPage"
      :page.sync="page"
      @page-count="pageCount = $event"
      hide-default-footer
      show-expand
      @click:row="onClickRow"
      mobile-breakpoint="0"
    >

      <template v-slot:expanded-item="{ headers, item }">
        <td :colspan="headers.length" class="pa-4">
          <Monster class="column-count-2" :monster="item" />
        </td>
      </template>

      <template v-slot:item.isInBestiary="{ item }">
        <v-simple-checkbox off-icon="mdi-bookmark-outline" on-icon="mdi-bookmark" @input="toggleMonsterInBestiary(item)" :value="isMonsterInBestiary(item)"></v-simple-checkbox>
      </template>

      <template v-slot:item.title="{ item }">
        <router-link :to="{ path: item.path }" class="subtitle-2">{{ item.title }}</router-link>
      </template>

      <template v-slot:item.frontmatter.challenge="{ item }">
        <span>{{ displayChallenge(item.frontmatter.challenge) }}</span>
      </template>

      <template v-slot:item.frontmatter.environments="{ item }">
        <span v-if="item.frontmatter.environments">{{ displayList(item.frontmatter.environments) }}</span>
      </template>

      <template v-slot:item.frontmatter.dungeonTypes="{ item }">
        <span v-if="item.frontmatter.dungeonTypes">{{ displayList(item.frontmatter.dungeonTypes) }}</span>
      </template>

      <template v-slot:item.isInEncounter="{ item }">
        <v-tooltip top>
          <template v-slot:activator="{ on, attrs }">
            <v-btn
              dense
              icon
              @click.stop="addCreatureInEncounter(item)"
              v-bind="attrs"
              v-on="on">
              <v-icon>mdi-sword-cross</v-icon>
            </v-btn>
          </template>
          <span>Ajouter au calculateur de rencontre</span>
        </v-tooltip>
      </template>

    </v-data-table>

    <v-row class="align-center  mb-12 pb-6">
      <v-col :cols="12" md="3">
        <v-select v-model="itemsPerPage" :items="itemsPerPageChoices" label="Lignes par page" @change="selectItemPerPage"></v-select>
      </v-col>
      <v-col :cols="12" md="9">
        <v-pagination v-model="page" :length="pageCount" :total-visible="7" @input="changePage"></v-pagination>
      </v-col>
    </v-row>
  </div>
</template>

<script>
import { mapState } from 'vuex'
import Breadcrumb from '@theme/components/Breadcrumb'
import {
  displayChallenge,
  displayAC,
  displayHP,
  displayMovement,
  displaySavingThrows,
  displaySkills,
  displayVulnerabilities,
  displayResistances,
  displayImmunities,
  displayConditionImmunities,
  displaySenses,
} from '@theme/util/monsterHelpers'
import { setUrlParams, getUrlParameter } from '@theme/util/filterHelpers'
import { isResourceInLibrary, handleTooltips } from '@theme/util'
import Monster from '@theme/components/Monster'
import MyMonstersButton from '@theme/global-components/MyMonstersButton'
import { CHALLENGES } from '../../data/monsters'
import Cookies from 'js-cookie'

export default {
  components: { Breadcrumb, Monster, MyMonstersButton },

  data () {
    return {
      sortBy: 'title',
      sortDesc: false,
      itemsPerPage: 10,
      itemsPerPageChoices: [
        {text: '5', value: 5},
        {text: '10', value: 10},
        {text: '15', value: 15},
        {text: 'Tous', value: -1},
      ],
      page: 1,
      pageCount: 0,
      showColumn: {
        type: true,
        size: true,
        subtype: false,
        environments: false,
        dungeonTypes: false,
        encounter: true,
      },
      challenges: CHALLENGES
    }
  },

  computed: {
    ...mapState({
      search: state => state.monsterFilters.search,
      challengeRange: state => state.monsterFilters.challengeRange,
      types: state => state.monsterFilters.types,
      sizes: state => state.monsterFilters.sizes,
      environments: state => state.monsterFilters.environments,
      dungeonTypes: state => state.monsterFilters.dungeonTypes,
      speedFly: state => state.monsterFilters.speedFly,
      speedSwim: state => state.monsterFilters.speedSwim,
      speedBurrow: state => state.monsterFilters.speedBurrow,
    }),

    headers() {
      let headers = [
        { text: "", align: 'center', sortable: false, value: 'isInBestiary' },
        { text: "Nom", align: 'start', sortable: true, value: 'title' },
        { text: "ID", align: 'center', sortable: true, value: 'frontmatter.challenge' },
      ]
      if (this.showColumn.type && this.$vuetify.breakpoint.mdAndUp) {
        headers.push({ text: "Type", align: 'start', sortable: false, value: 'frontmatter.type' })
      }
      if (this.showColumn.size && this.$vuetify.breakpoint.mdAndUp) {
        headers.push({ text: "Taille", align: 'center', sortable: false, value: 'frontmatter.size' })
      }
      if (this.showColumn.subtype && this.$vuetify.breakpoint.mdAndUp) {
        headers.push({ text: "Sous-type", align: 'start', sortable: false, value: 'frontmatter.subtype' })
      }
      if (this.showColumn.environments && this.$vuetify.breakpoint.mdAndUp) {
        headers.push({ text: "Environnements", align: 'start', sortable: false, value: 'frontmatter.environments' })
      }
      if (this.showColumn.dungeonTypes && this.$vuetify.breakpoint.mdAndUp) {
        headers.push({ text: "Type de donjons", align: 'start', sortable: false, value: 'frontmatter.dungeonTypes' })
      }
      if (this.showColumn.encounter && this.$vuetify.breakpoint.mdAndUp) {
        headers.push({ text: "Rencontre", align: 'center', sortable: false, value: 'isInEncounter' })
      }
      return headers
    },

    selectedTypes() {
      let result = []
      for (let type of this.types) {
        if (type.value) {
          result.push(type.label)
        }
      }
      return result
    },

    selectedSizes() {
      let result = []
      for (let size of this.sizes) {
        if (size.value) {
          result.push(size.label)
        }
      }
      return result
    },

    selectedEnvironments() {
      let result = []
      for (let env of this.environments) {
        if (env.value) {
          result.push(env.label)
        }
      }
      return result
    },

    selectedDungeonTypes() {
      let result = []
      for (let dType of this.dungeonTypes) {
        if (dType.value) {
          result.push(dType.label)
        }
      }
      return result
    },

    monsters() {
      let results = this.$pagination.pages

      // Filter ID
      let minID = this.challengeRange[0]
      let maxID = this.challengeRange[1]
      if (this.challengeRange[0] > this.challengeRange[1]) {
        minID = this.challengeRange[1]
        maxID = this.challengeRange[0]
      }
      results = results.filter(item => {
        return item.frontmatter.challenge >= Number(CHALLENGES[minID].value) && item.frontmatter.challenge <= Number(CHALLENGES[maxID].value)
      })

      // Filter types
      let selectedTypes = []
      for (var i = 0; i < this.types.length; i++) {
        if (this.types[i].value) {
          selectedTypes.push(this.types[i].label)
        }
      }
      if (selectedTypes.length) {
        results = results.filter(item => {
          return selectedTypes.indexOf(item.frontmatter.type) > -1
        })
      }

      // Filter sizes
      let selectedSizes = []
      for (var i = 0; i < this.sizes.length; i++) {
        if (this.sizes[i].value) {
          selectedSizes.push(this.sizes[i].abbr)
        }
      }
      if (selectedSizes.length) {
        results = results.filter(item => {
          return selectedSizes.indexOf(item.frontmatter.size) > -1
        })
      }

      // Filter environments
      let selectedEnvironments = []
      for (var i = 0; i < this.environments.length; i++) {
        if (this.environments[i].value) {
          selectedEnvironments.push(this.environments[i].label)
        }
      }
      if (selectedEnvironments.length) {
        let classFiltered = []
        for (var i = 0; i < selectedEnvironments.length; i++) {
          for (var j = 0; j < results.length; j++) {
            if (results[j].frontmatter.environments) {
              if (results[j].frontmatter.environments.indexOf(selectedEnvironments[i]) > -1) {
                if (classFiltered.indexOf(results[j]) < 0) {
                  classFiltered.push(results[j])
                }
              }
            }
          }
        }
        results = classFiltered
      }

      // Filter dungeon types
      let selectedDungeonTypes = []
      for (var i = 0; i < this.dungeonTypes.length; i++) {
        if (this.dungeonTypes[i].value) {
          selectedDungeonTypes.push(this.dungeonTypes[i].label)
        }
      }
      if (selectedDungeonTypes.length) {
        let classFiltered = []
        for (var i = 0; i < selectedDungeonTypes.length; i++) {
          for (var j = 0; j < results.length; j++) {
            if (results[j].frontmatter.dungeonTypes) {
              if (results[j].frontmatter.dungeonTypes.indexOf(selectedDungeonTypes[i]) > -1) {
                if (classFiltered.indexOf(results[j]) < 0) {
                  classFiltered.push(results[j])
                }
              }
            }
          }
        }
        results = classFiltered
      }

      // Filter movement speeds
      if (this.speedFly !== undefined) {
        if (this.speedFly === true) {
          results = results.filter(item => {
            return item.frontmatter.movement.fly > 0
          })
        } else {
          results = results.filter(item => {
            return item.frontmatter.movement.fly == null
          })
        }
      }
      if (this.speedSwim !== undefined) {
        if (this.speedSwim === true) {
          results = results.filter(item => {
            return item.frontmatter.movement.swim > 0
          })
        } else {
          results = results.filter(item => {
            return item.frontmatter.movement.swim == null
          })
        }
      }
      if (this.speedBurrow !== undefined) {
        if (this.speedBurrow === true) {
          results = results.filter(item => {
            return item.frontmatter.movement.burrow > 0
          })
        } else {
          results = results.filter(item => {
            return item.frontmatter.movement.burrow == null
          })
        }
      }

      // let json = []
      // for (var monster of results) {
      //   let m = {}
      //   m.name = monster.frontmatter.title
      //   m.size = monster.frontmatter.size
      //   m.alignment = monster.frontmatter.alignment
      //   m.type = monster.frontmatter.type
      //   m.subtype = monster.frontmatter.subtype
      //   m.swarm = monster.frontmatter.isSwarm
      //   m.challenge = monster.frontmatter.challenge
      //   m.ac = displayAC(monster)
      //   m.hp = displayHP(monster)
      //   m.speed = displayMovement(monster)
      //   m.abilityScores = monster.frontmatter.abilityScores
      //   m.savingThrows = displaySavingThrows(monster)
      //   m.skills = displaySkills(monster)
      //   m.vulnerabilities = displayVulnerabilities(monster)
      //   m.resistances = displayResistances(monster)
      //   m.immunities = displayImmunities(monster)
      //   m.conditionImmunities = displayConditionImmunities(monster)
      //   m.senses = displaySenses(monster)
      //   m.languages = monster.frontmatter.languages
      //   m.telepathy = monster.frontmatter.telepathy
      //   m.source = monster.frontmatter.source
      //   m.sourcePage = monster.frontmatter.source_page
      //   m.content = monster.rawContent
      //   m.environments = monster.frontmatter.environments
      //   m.dungeon_types = monster.frontmatter.dungeonTypes
      //   json.push(m)
      // }
      //
      // console.log(json)
      // console.log(results)

      return results
    }
  },

  methods: {
    displayList (list) { return list.join(', ') },
    displayChallenge (challenge) { return displayChallenge(challenge) },
    isMonsterInBestiary (monster) {
      return isResourceInLibrary(monster, this.$store.state.myMonsters.monsters)
    },
    toggleMonsterInBestiary (monster) {
      if (this.isMonsterInBestiary(monster)) {
        this.$store.commit('myMonsters/removeMonster', monster)
        this.$store.commit('setSnackbarText', "Le monstre " + monster.title + " a été supprimé de votre bestiaire")
        this.$store.commit('setIsOpenSnackbar', true)
      } else {
        this.$store.commit('myMonsters/addMonster', monster)
        this.$store.commit('setSnackbarText', "Le monstre " + monster.title + " a été ajouté à votre bestiaire")
        this.$store.commit('setIsOpenSnackbar', true)
      }
    },

    isCreatureInEncounter (creature) {
      return isResourceInLibrary(creature, this.$store.state.encounterCalculator.creatures)
    },
    addCreatureInEncounter (creature) {
      this.$store.commit('encounterCalculator/addCreature', creature)
    },

    selectItemPerPage (value) {
      setUrlParams("lignes", [value])
      let self = this
      setTimeout(function () {
        handleTooltips({pages:self.$site.pages})
      }, 100);
    },

    changePage (page) {
      // console.log(page)
      setUrlParams("page", [page])
      let self = this
      setTimeout(function () {
        handleTooltips({pages:self.$site.pages})
      }, 100);
    },

    onClickRow (row, item) {
      item.expand(!item.isExpanded)
    },

    setShowColumn () {
      Cookies.set('5e-drs-bestiaire-colonnes', this.showColumn, { expires: 365 })
    },
  },

  watch: {
    monsters: function (newMonsters, oldMonsters) {
      let self = this
      setTimeout(function () {
        handleTooltips({pages:self.$site.pages})
      }, 100);
    }
  },

  mounted () {
    this.$store.commit('setHasRightDrawer', true)
    this.$store.commit('setRightDrawer', this.$vuetify.breakpoint.lgAndUp)
    this.$store.commit('setInRightDrawer', 'monsterFilters')

    let itemsPerPage = parseInt(getUrlParameter(window.location.href, "lignes"))
    if (itemsPerPage) {
      this.itemsPerPage = itemsPerPage
    }
    let page = parseInt(getUrlParameter(window.location.href, "page"))
    if (page) {
      this.page = page
    }

    const showColumn = Cookies.get('5e-drs-bestiaire-colonnes')
    if (showColumn) {
      this.showColumn = JSON.parse(showColumn)
    }

    let self = this
    setTimeout(function () {
      handleTooltips({pages:self.$site.pages})
    }, 100);
    this.$router.afterEach(() => {
      setTimeout(function () {
        handleTooltips({pages:self.$site.pages})
      }, 100)
    })
  }
}
</script>

<style lang="scss">

</style>
