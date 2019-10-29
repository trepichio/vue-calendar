<template>
  <v-row class="fill-height">
    <v-col>
      <v-sheet height="64">
        <v-toolbar flat color="white">
          <v-btn color="secondary" dark class="mr-4" @click="dialog = true">Novo Evento</v-btn>
          <v-btn outlined class="mr-4" @click="setToday">Hoje</v-btn>
          <v-btn fab text small @click="prev">
            <v-icon small>mdi-chevron-left</v-icon>
          </v-btn>
          <v-btn fab text small @click="next">
            <v-icon small>mdi-chevron-right</v-icon>
          </v-btn>
          <v-toolbar-title>{{ title }}</v-toolbar-title>
          <div class="flex-grow-1"></div>

          <v-menu bottom right>
            <template v-slot:activator="{ on }">
              <v-btn outlined v-on="on">
                <span>{{ typeToLabel[type] }}</span>
                <v-icon right>mdi-menu-down</v-icon>
              </v-btn>
            </template>
            <v-list>
              <v-list-item @click="type = 'day'">
                <v-list-item-title>Dia</v-list-item-title>
              </v-list-item>
              <v-list-item @click="type = 'week'">
                <v-list-item-title>Semana</v-list-item-title>
              </v-list-item>
              <v-list-item @click="type = 'month'">
                <v-list-item-title>Mês</v-list-item-title>
              </v-list-item>
              <v-list-item @click="type = '4day'">
                <v-list-item-title>4 dias</v-list-item-title>
              </v-list-item>
            </v-list>
          </v-menu>
        </v-toolbar>
      </v-sheet>

      <!-- Add Event dialog -->
      <v-dialog v-model="dialog" max-width="500">
        <v-card>
          <v-container>
            <v-alert dense v-if="!valid" type="error">Há campos obrigatórios não preenchidos.</v-alert>
            <v-form ref="formAdd" v-model="valid" lazy-validation>
              <v-text-field
                required
                :rules="nameRules"
                v-model="name"
                type="text"
                label="Nome do evento (obrigatório)"
              ></v-text-field>
              <v-text-field v-model="details" type="text" label="Detalhes do evento"></v-text-field>
              <v-row justify="center">
                <v-date-picker
                  locale="pt"
                  :selected-items-text="`${dateRangeText}`"
                  v-model="dates"
                  @input="setDates"
                  :range="range"
                  :events="functionEvents"
                  :min="minDays"
                  :max="addDays(minDays,maxDays)"
                  :allowed-dates="allowedDates"
                ></v-date-picker>
                <v-text-field
                  required
                  :rules="dateRules"
                  class="mt-3"
                  v-model="dateRangeText"
                  label="Intervalo de datas (selecione acima)"
                  prepend-icon="mdi-event"
                  readonly
                ></v-text-field>
              </v-row>
              <v-row justify="center" v-if="dates.length === 1 || dates[0] === dates[1]">
                <editTimeDialog ref="timeDialog" />
              </v-row>
              <v-row justify="center">
                <v-color-picker
                  class="ma-2"
                  show-swatches
                  dot-size="20"
                  hide-inputs
                  hide-mode-switch
                  flat
                  swatches-max-height="100px"
                  v-model="color"
                ></v-color-picker>
              </v-row>
              <v-btn
                color="primary"
                class="mr-4"
                :disabled="!valid"
                @click.stop="addEvent"
              >Adicionar</v-btn>
            </v-form>
          </v-container>
        </v-card>
      </v-dialog>
      <v-sheet height="600">
        <v-calendar
          ref="calendar"
          v-model="focus"
          color="primary"
          :events="events"
          :event-color="getEventColor"
          :event-margin-bottom="3"
          :now="today"
          :type="type"
          :locale="'pt'"
          @click:event="showEvent"
          @click:more="viewDay"
          @click:date="viewDay"
          @change="updateRange"
          event-more-text="Mais {0}..."
          :interval-format="val => val.time"
        ></v-calendar>
        <v-menu
          v-model="selectedOpen"
          :close-on-content-click="false"
          :activator="selectedElement"
          offset-x
        >
          <v-card color="grey lighten-4" min-width="350px" flat>
            <v-toolbar :color="selectedEvent.color" dark>
              <v-btn icon>
                <v-icon @click="deleteEvent(selectedEvent.id)">mdi-delete</v-icon>
              </v-btn>
              <v-toolbar-title v-html="selectedEvent.name"></v-toolbar-title>
              <div class="flex-grow-1"></div>
            </v-toolbar>
            <v-card-text>
              <form v-if="currentlyEditing !== selectedEvent.id">{{ selectedEvent.details }}</form>
              <form v-else>
                <textarea-autosize
                  v-model="selectedEvent.details"
                  type="text"
                  style="width: 100%"
                  :min-height="100"
                  placeholder="adicione uma descrição"
                ></textarea-autosize>
              </form>
            </v-card-text>
            <v-card-actions>
              <v-btn text color="secondary" @click="selectedOpen = false">Close</v-btn>
              <v-btn
                text
                v-if="currentlyEditing !== selectedEvent.id"
                @click.prevent="editEvent(selectedEvent)"
              >Edit</v-btn>
              <v-btn text v-else @click.prevent="updateEvent(selectedEvent)">Save</v-btn>
            </v-card-actions>
          </v-card>
        </v-menu>
      </v-sheet>

      <v-sheet height="auto" dark v-if="user.admin">
        <v-row class="text-center text-sm-right">
          <v-col cols="12" xs="12" sm="6" md="6">
            <v-row>
              <v-col cols="12" md="6" class="d-flex align-center justify-center headline">
                <span>Nº máx compromissos:</span>
              </v-col>
              <v-col cols="12" md="6" class="text-center">
                <v-btn
                  @click.prevent="incrementBusy"
                  fab
                  x-small
                  color=" red lighten-4"
                  class="mx-3 black--text"
                >
                  <v-icon>mdi-plus</v-icon>
                </v-btn>
                <span class="quantifiers">{{ amountIsBusy | padNumber }}</span>
                <v-btn
                  @click.prevent="decrementBusy"
                  fab
                  x-small
                  color=" red lighten-4"
                  class="mx-3 black--text"
                >
                  <v-icon>mdi-minus</v-icon>
                </v-btn>
                <div>
                  <span>por dia</span>
                </div>
              </v-col>
            </v-row>
          </v-col>
          <v-col cols="12" xs="12" sm="6" md="6">
            <v-row>
              <v-col cols="12" class="d-flex align-center justify-center headline">
                <span>Permitir agendar até:</span>
              </v-col>
              <v-col cols="12" class="text-center">
                <v-btn
                  @click.prevent="incrementMaxDays"
                  fab
                  x-small
                  color=" red lighten-4"
                  class="mx-3 black--text"
                >
                  <v-icon>mdi-plus</v-icon>
                </v-btn>
                <span class="quantifiers">{{ maxDays | padNumber }}</span>
                <v-btn
                  @click.prevent="decrementMaxDays"
                  fab
                  x-small
                  color=" red lighten-4"
                  class="mx-3 black--text"
                >
                  <v-icon>mdi-minus</v-icon>
                </v-btn>
                <div class="text-center">
                  <span>dias</span>
                </div>
              </v-col>
            </v-row>
          </v-col>
        </v-row>
      </v-sheet>
    </v-col>
  </v-row>
</template>

<script>
import { db } from "@/main";
import editTimeDialog from "@/components/editTimeDialog.vue";

export default {
  name: "Calendar",
  components: {
    editTimeDialog
  },
  data: () => ({
    today: new Date().toISOString().substr(0, 10),
    focus: new Date().toISOString().substr(0, 10),
    type: "month",
    typeToLabel: {
      month: "Mês",
      week: "Semana",
      day: "Dia",
      "4day": "4 Dias"
    },
    minDays: new Date().toISOString().substr(0, 10),
    maxDays: 40,
    dates: [],
    start: null,
    end: null,
    selectedEvent: {},
    selectedElement: null,
    selectedOpen: false,
    events: [],
    name: "",
    details: "",
    // eslint-disable-next-line no-dupe-keys
    start: null,
    // eslint-disable-next-line no-dupe-keys
    end: null,
    color: "#1976D2",
    currentlyEditing: null,
    // eslint-disable-next-line no-dupe-keys
    selectedEvent: {},
    // eslint-disable-next-line no-dupe-keys
    selectedOpen: false,
    dialog: false,
    valid: null,
    amountIsBusy: 3,
    nameRules: [
      v => (!!v && v.trim() !== "") || "Este campo não deve ficar em branco."
    ],
    dateRules: [
      v => (!!v && v.length > 0) || "Ao menos uma data deve ser selecionada."
    ],
    appointments: [],
    user: {
      admin: false
    },
    range: true
  }),
  filters: {
    padNumber(number) {
      return String(number).padStart(2, "0");
    }
  },
  computed: {
    busyDaysOnly() {
      return this.appointments
        .filter(([, count]) => count >= this.amountIsBusy)
        .map(([day]) => day);
    },
    functionEvents() {
      return this.dateFunctionEvents;
    },
    dateRangeText() {
      return this.dates
        .map(
          dString =>
            `${new Date(dString).getUTCDate()}/${new Date(
              dString
            ).getUTCMonth()}`
        )
        .join(" a ");
    },
    title() {
      const { start, end } = this;
      if (!start || !end) {
        return "";
      }

      const startMonth = this.monthFormatter(start);
      const endMonth = this.monthFormatter(end);
      const formattedStartMonth =
        startMonth[0].toUpperCase() + startMonth.slice(1);
      const formattedEndMonth = endMonth[0].toUpperCase() + endMonth.slice(1);

      const suffixMonth =
        formattedStartMonth === formattedEndMonth
          ? ""
          : "de " + formattedEndMonth;

      const startYear = start.year;
      const endYear = end.year;
      const suffixYear = startYear === endYear ? "" : "de " + endYear;

      const startDay = start.day; // + this.nth(start.day)
      const endDay = end.day; // + this.nth(end.day)

      switch (this.type) {
        case "month":
          return `${formattedStartMonth} de ${startYear}`;
        case "week":
        case "4day":
          return `${startDay} de ${formattedStartMonth} de ${startYear} a dia ${endDay} ${suffixMonth} ${suffixYear}`;
        case "day":
          return `${startDay} de ${formattedStartMonth} de ${startYear}`;
      }
      return "";
    },
    monthFormatter() {
      return this.$refs.calendar.getFormatter({
        timeZone: "UTC",
        month: "long"
      });
    }
  },
  watch: {
    events(val) {
      let dates = [];

      val.forEach(evt => {
        dates.push(...this.calcDates(evt));
      });
      this.appointments = this.countDays(dates);
    }
  },
  mounted() {
    this.$refs.calendar.checkChange();
    this.getEvents();
  },
  methods: {
    setDates(val) {
      val.sort((a, b) => new Date(a) - new Date(b));
    },
    incrementBusy() {
      this.amountIsBusy++;
    },
    decrementBusy() {
      this.amountIsBusy--;
    },
    incrementMaxDays() {
      this.maxDays++;
    },
    decrementMaxDays() {
      this.maxDays--;
    },
    updateBusyDays(val = this.amountIsBusy) {
      this.busyDaysOnly = this.appointments
        .filter(([count]) => count >= val)
        .map(([day]) => day);
    },
    calcDates(evt) {
      let aDates = [];
      const [sStartNotime, sEndNotime] = [
        evt.start.substr(0, 10),
        evt.end.substr(0, 10)
      ];
      const [start, end] = [new Date(sStartNotime), new Date(sEndNotime)];
      const diff = (end - start) / 86400000;
      // console.warn(`"diff btw ${sStartNotime} and ${sEndNotime}"`, diff)

      for (var i = 0; i <= diff; i++) {
        let next = new Date(start);
        const betweenDate = new Date(next.setUTCDate(start.getUTCDate() + i))
          .toISOString()
          .substr(0, 10);

        // console.warn("betweenDate", betweenDate)
        aDates.push(betweenDate);
      }
      // console.warn("aDates", aDates)
      return aDates;
    },
    editEvent(ev) {
      this.currentlyEditing = ev.id;
    },
    addDays(date, days) {
      let result = new Date(date);
      result.setDate(result.getDate() + days);
      return result.toISOString().substr(0, 10);
    },
    weekdays: val => [1, 2, 3, 4, 5].includes(new Date(val).getUTCDay()),
    busydays: (val, bd) =>
      bd.includes(new Date(val).toISOString().substr(0, 10)),
    allowedDates: function(val) {
      return this.weekdays(val) && !this.busydays(val, this.busyDaysOnly);
    },
    dateFunctionEvents(date) {
      if (this.busyDaysOnly.includes(date)) return true;
      return false;
    },
    countDays(days) {
      const aCount = [...new Set(days)].map(x => [
        x,
        days.filter(y => y === x).length
      ]);
      return aCount;
    },
    async addEvent() {
      let timeDialog = this.$refs.timeDialog;
      if (this.$refs.formAdd.validate()) {
        let [start, end = start] = [...this.dates];
        if (timeDialog.time) start += ` ${timeDialog.time}`;
        await db.collection("calEvent").add({
          name: this.name.trim(),
          details: this.details.trim(),
          start: start,
          end: end,
          color: this.color
        });
        this.getEvents();
        this.name = "";
        this.details = "";
        this.start = "";
        this.end = "";
        this.color = "";
        this.dates = [];
        this.$refs.timeDialog.time = "";
        this.dialog = false;
        this.$refs.formAdd.resetValidation();
      } else {
        this.dialog = true;
      }
    },
    async updateEvent(ev) {
      await db
        .collection("calEvent")
        .doc(this.currentlyEditing)
        .update({
          details: ev.details
        });
      this.selectedOpen = false;
      this.currentlyEditing = null;
    },
    async deleteEvent(ev) {
      await db
        .collection("calEvent")
        .doc(ev)
        .delete();
      this.selectedOpen = false;
      this.getEvents();
    },
    async getEvents() {
      let snapshot = await db.collection("calEvent").get();
      let events = [];

      snapshot.forEach(doc => {
        let appData = doc.data();
        appData.id = doc.id;
        events.push(appData);
      });

      this.events = events;
    },
    viewDay({ date }) {
      this.focus = date;
      this.type = "day";
    },
    getEventColor(event) {
      return event.color;
    },
    setToday() {
      this.focus = this.today;
    },
    prev() {
      this.$refs.calendar.prev();
    },
    next() {
      this.$refs.calendar.next();
    },
    showEvent({ nativeEvent, event }) {
      const open = () => {
        this.selectedEvent = event;
        this.selectedElement = nativeEvent.target;
        setTimeout(() => (this.selectedOpen = true), 10);
      };

      if (this.selectedOpen) {
        this.selectedOpen = false;
        setTimeout(open, 10);
      } else {
        open();
      }

      nativeEvent.stopPropagation();
    },
    updateRange({ start, end }) {
      // You could load events from an outside source (like database) now that we have the start and end dates on the calendar
      this.start = start;
      this.end = end;
    },
    // a helper function to format dates in English locale
    nth(d) {
      return d > 3 && d < 21
        ? "th"
        : ["th", "st", "nd", "rd", "th", "th", "th", "th", "th", "th"][d % 10];
    }
  }
};
</script>

<style lang="css" scoped>
.quantifiers {
  color: red;
  font-size: 2rem;
  vertical-align: middle;
}
</style>