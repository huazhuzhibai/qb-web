<template>
  <v-dialog
    :value="true"
    @input="closeDialog"
    :fullscreen="phoneLayout"
    persistent
    width="40em"
  >
    <v-card>
      <v-card-title
        class="headline grey lighten-4"
      >
        <v-icon class="mr-2">mdi-server</v-icon>
        <span>Edit tracker</span>
      </v-card-title>
      <v-card-text class="pa-0">
        <v-stepper v-model="step">
          <v-stepper-header>
            <v-stepper-step :complete="step > 1" step="1">Search</v-stepper-step>
            <v-divider />
            <v-stepper-step :complete="step > 2" step="2">Preview</v-stepper-step>
            <v-divider />
            <v-stepper-step :complete="step > 3" step="3">Result</v-stepper-step>
          </v-stepper-header>
          <v-stepper-items>
            <v-stepper-content step="1">
              <v-form v-model="valid">
                <v-text-field
                  v-model="search"
                  label="Search"
                  :rules="[v => !!v || 'Required']"
                  placeholder="Regex format"
                  required
                />
                <v-text-field
                  v-model="replace"
                  label="Replace"
                />
              </v-form>
            </v-stepper-content>
            <v-stepper-content step="2">
              {{ toEdit.length }} torrent(s) to update.
              <ol class="torrents pt-6">
                <li v-for="(row, i) in toEdit" :key="i">
                  {{ row.name }}
                  <br>
                  {{ row.origUrl }}
                  <br>
                  {{ row.newUrl }}
                </li>
              </ol>
            </v-stepper-content>
            <v-stepper-content step="3">
              <v-progress-linear
                v-if="submitting && currentIndex != toEdit.length"
                :value="currentIndex / toEdit.length * 100"
              />
              <template v-else>
                {{ currentIndex }} torrent(s) updated.
              </template>
            </v-stepper-content>
          </v-stepper-items>
        </v-stepper>
      </v-card-text>
      <v-card-actions>
        <v-spacer />
        <v-btn
          text
          @click="back"
          v-if="step < 3"
          v-text="step == 1 ? 'Cancel' : 'Back'"
        >Back</v-btn>
        <v-btn
          @click="foward"
          color="warning"
          :disabled="!canNext"
          :loading="submitting"
          v-text="[null, 'Next', 'Confirm', 'Close'][step]"
        />
      </v-card-actions>
    </v-card>
  </v-dialog>
</template>

<script lang="ts">
import _ from 'lodash';
import Vue from 'vue';
import api from '@/Api';
import { mapGetters } from 'vuex';

export default Vue.extend({
  props: {
    value: Array,
  },
  data() {
    return {
      step: 1,
      valid: false,
      submitting: false,
      torrents: [],
      search: '',
      replace: '',
      toEdit: [],
      currentIndex: 0,
    };
  },
  created() {
    this.torrents = this.value;
  },
  computed: {
    ...mapGetters(['allTorrents']),
    phoneLayout() {
      return this.$vuetify.breakpoint.xsOnly;
    },
    canNext() {
      if (this.step == 1 && this.valid) {
        return true;
      } else if (this.step == 2 && this.toEdit.length > 0) {
        return true;
      } else if (this.step == 3 && !this.submitting) {
        return true;
      } else {
        return false;
      }
    },
  },
  methods: {
    closeDialog() {
      this.$emit('input', []);
    },
    calcResults() {
      const regex = new RegExp(this.search);

      return _.chain(this.torrents)
        .map(({tracker, hash, name}) => {
          const newUrl = tracker.replace(regex, this.replace);
          return newUrl == tracker ? null : {
            hash,
            name,
            origUrl: tracker,
            newUrl,
          };
        }).compact().value();
    },
    back() {
      if (this.step == 1) {
        this.closeDialog();
        return;
      }
      this.step--;
    },
    async foward() {
      if (this.step == 1) {
        this.toEdit = this.calcResults();
        this.step++;
        return;
      }
      if (this.step == 3) {
        this.closeDialog();
        return;
      }

      if (this.submitting) {
        return;
      }

      this.submitting = true;
      this.step++;

      this.currentIndex = 0;

      for (const item of this.toEdit) {
        await api.editTracker(item.hash, item.origUrl, item.newUrl);
        this.currentIndex++;
      }

      this.submitting = false;
    },
  },
});
</script>

<style lang="scss" scoped>
.torrents {
  overflow: auto;
  white-space: nowrap;
}

.v-stepper {
  box-shadow: none;
}

.v-dialog--fullscreen {
  .v-card__text {
    padding-bottom: 52px;
  }

  .v-card__actions {
    position: absolute;
    bottom: 0;
    right: 0;
  }
}
</style>