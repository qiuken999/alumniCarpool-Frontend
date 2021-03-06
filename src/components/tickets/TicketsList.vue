<template>
  <div>
    <h4 class="d-flex justify-content-between align-items-center w-100 font-weight-bold py-3 mb-4">
      <div>Tickets</div>
      <router-link v-if="currentUser.currentRole==='Driver'" :to="{ name: 'tickets-create'}"><b-btn variant="primary" class="d-block"><span class="ion ion-md-add"></span>&nbsp; Create Ticket</b-btn></router-link>
    </h4>

    <!-- Filters -->
    <div class="ui-bordered px-4 pt-4 mb-4">
      <div class="form-row align-items-center">
        <div class="col-md mb-4">
          <label class="form-label">Departure City</label>
          <b-select v-model="departure_from" :options="['Windsor', 'Waterloo', 'Toronto', 'Markham']" />
        </div>
        <div class="col-md mb-4">
          <label class="form-label">Arrival City</label>
          <b-select v-model="departure_to" :options="['Windsor', 'Waterloo', 'Toronto', 'Markham']" />
        </div>
        <div class="col-md mb-4">
          <label class="form-label">Departure Date</label>
          <flat-pickr v-model="departureDate" :config="{ altInput: true, animate: !isRTL, dateFormat: 'm/d/Y', altFormat: 'm/d/Y', mode: 'single' }" :placeholder="!isIEMode ? 'Select a date' : null" />
        </div>
        <div class="col-md col-xl-2 mb-4">
          <label class="form-label d-none d-md-block">&nbsp;</label>
          <b-btn variant="primary" :block="true" @click="show(departure_from, departure_to, departureDate)">Show</b-btn>
        </div>
      </div>
    </div>
    <!-- / Filters -->

    <b-card no-body>

      <!-- Table controls -->
      <b-card-body>

        <div class="row">
          <div class="col">
            Per Page: &nbsp;
            <b-select size="sm" v-model="perPage" :options="[10, 20, 30, 40, 50]" class="d-inline-block w-auto" />
          </div>
          <div class="col">
            <b-input size="sm" placeholder="Search..." class="d-inline-block w-auto float-sm-right" @input="filter($event)" />
          </div>
        </div>

      </b-card-body>
      <!-- / Table controls -->

      <!-- Table -->
      <hr class="border-light m-0">
      <div class="table-responsive">
        <b-table
          :items="ticketsData"
          :fields="(currentUser.currentRole==='Passenger') ? fields_passenger : fields_driver"
          :sort-by.sync="sortBy"
          :sort-desc.sync="sortDesc"
          :striped="true"
          :bordered="true"
          :current-page="currentPage"
          :per-page="perPage"
          class="card-table">

          <template slot="driver" slot-scope="data">
            <div v-if="data.value" class="ui-feed-icon-container d-inline-block mr-2">
              <img :src="getImage(data.value.id, 'avatar')" @click.stop="showModal_avatar(data.value)" v-b-tooltip.hover :title="data.value.firstName+' '+data.value.lastName" class="d-block ui-w-30 rounded-circle">
            </div>
          </template>

          <template slot="car" slot-scope="data">
            <div v-if="data.item.driver.car" class="ui-feed-icon-container d-inline-block mr-2">
              <img :src="getImage(data.item.driver.id, 'car')" @click.stop="showModal_car(data.item.driver)" v-b-tooltip.hover :title="'Plate number: '+data.item.driver.car.plateNum" class="d-block ui-w-30 rounded-circle">
            </div>
          </template>

          <template slot="certificate" slot-scope="data">
            <div v-if="data.item.driver" class="ui-feed-icon-container d-inline-block mr-2">
              <img :src="getImage(data.item.driver.id, 'certificate')" @click.stop="showModal_certificate(data.item.driver)" v-b-tooltip.hover :title="'Alma mater: '+data.item.driver.almaMater" class="d-block ui-w-30 rounded-circle">
            </div>
          </template>


          <template slot="seats" slot-scope="data">
              <!-- <b-btn v-for="seat in seats" variant="default btn-xs icon-btn md-btn-flat" v-b-tooltip.hover title="Occupy" @click.stop="occupySeat()"><i class="ion ion-md-person-add"></i></b-btn> -->
              <div v-for="(seat, i) in data.item.driver.car.seats" :key="i" class="ui-feed-icon-container d-inline-block mr-2">
                  <!-- <a @click.prevent="seats.splice(i, 1)" href="#" class="ui-icon ui-feed-icon ion ion-ios-close bg-secondary text-white"></a> -->
                  <!-- <img :src="`${baseUrl}img/avatars/1.png`" v-b-tooltip.hover :title="seat" class="ticket-assignee d-block ui-w-35 rounded-circle"> -->
                  <!-- <a href="javascript:void(0)" class="far fa-user ticket-assignee-add d-block ui-w-35 rounded-circle"></a> -->
                  <b-btn v-if="seat.reserved" variant="outline-success btn-sm icon-btn md-btn-flat" v-b-tooltip.hover.bottom :title="seat.position+', reserved by '+seat.passenger.firstName+' '+seat.passenger.lastName" @click.stop="releaseSeat(data.item, i)"><i class="fas fa-user-alt"></i></b-btn>
                  <b-btn v-if="!seat.reserved" variant="outline-success btn-sm icon-btn md-btn-flat" v-b-tooltip.hover.bottom :title="'Position: '+seat.position" @click.stop="reserveSeat(data.item, i)"><i class="far fa-user"></i></b-btn>
              </div>
          </template>

          <template slot="action" slot-scope="data" v-if="data.item.driver.id===currentUser.id">
              <div class="ui-feed-icon-container d-inline-block mr-2" v-if="data.item.status==='Boarding'">
                  <b-btn variant="outline-warning btn-sm icon-btn md-btn-flat" v-b-tooltip.hover title="Depart" @click.stop="startJourney(data.item)"><i class="ion ion-md-paper-plane"></i></b-btn>
              </div>
              <div class="ui-feed-icon-container d-inline-block mr-2" v-if="data.item.status==='On-board'">
                  <b-btn variant="outline-warning btn-sm icon-btn md-btn-flat" v-b-tooltip.hover title="Arrive" @click.stop="finishJourney(data.item)"><i class="ion ion-ios-paper-plane"></i></b-btn>
              </div>
              <div class="ui-feed-icon-container d-inline-block mr-2">
                  <b-btn variant="outline-warning btn-sm icon-btn md-btn-flat" v-b-tooltip.hover title="Cancel" @click.stop="deleteTicket(data.item)"><i class="ion ion-md-trash"></i></b-btn>
              </div>
          </template>

        </b-table>
      </div>

      <!-- Pagination -->
      <b-card-body class="pt-0 pb-3">

        <div class="row">
          <div class="col-sm text-sm-left text-center pt-3">
            <span class="text-muted" v-if="totalItems">Page {{ currentPage }} of {{ totalPages }}</span>
          </div>
          <div class="col-sm pt-3">
            <b-pagination class="justify-content-center justify-content-sm-end m-0"
              v-if="totalItems"
              v-model="currentPage"
              :total-rows="totalItems"
              :per-page="perPage"
              size="sm" />
          </div>
        </div>

      </b-card-body>
      <!-- / Pagination -->

    </b-card>
  </div>
</template>

<style src="@/vendor/libs/vue-flatpickr-component/vue-flatpickr-component.scss" lang="scss"></style>



<script>
import flatPickr from 'vue-flatpickr-component'
import 'flatpickr/dist/flatpickr.css';
import 'flatpickr/dist/themes/material_blue.css';
import InputTag from 'vue-input-tag';
import swal from 'sweetalert2';
import '@/vendor/libs/Animation.css';



export default {
  name: 'ticket-list',
  metaInfo: {
    title: 'Ticket list'
  },
  components: {
    flatPickr,
    InputTag,
    swal,
  },
  data: () => ({
    // Options
    searchKeys: ['departureLocation', 'arrivalLocation', 'departureTime', 'status'],
    sortBy: 'status',
    sortDesc: false,
    perPage: 10,
    currentPage: 1,

    fields_passenger: [
      { key: 'departureLocation', sortable: true, thClass: 'text-nowrap', tdClass: 'align-middle py-3' },
      { key: 'arrivalLocation', sortable: true, thClass: 'text-nowrap', tdClass: 'align-middle py-3' },
      { key: 'departureTime', sortable: true, thClass: 'text-nowrap', tdClass: 'align-middle py-3' },
      { key: 'status', sortable: true, thClass: 'text-nowrap', tdClass: 'align-middle py-3' },
      { key: 'driver', label: 'Driver', thClass: 'text-nowrap', tdClass: 'text-nowrap align-middle text-center py-3' },
      { key: 'car', label: 'Car', thClass: 'text-nowrap', tdClass: 'text-nowrap align-middle text-center py-3' },
      { key: 'certificate', label: 'Certificate', thClass: 'text-nowrap', tdClass: 'text-nowrap align-middle text-center py-3' },
      { key: 'seats', label: 'Seats', thClass: 'text-nowrap', tdClass: 'text-nowrap align-middle text-center py-3' },
      // { key: 'action', label: 'Action', thClass: 'text-nowrap', tdClass: 'text-nowrap align-middle text-center py-3' }
    ],

    fields_driver: [
      { key: 'departureLocation', sortable: true, thClass: 'text-nowrap', tdClass: 'align-middle py-3' },
      { key: 'arrivalLocation', sortable: true, thClass: 'text-nowrap', tdClass: 'align-middle py-3' },
      { key: 'departureTime', sortable: true, thClass: 'text-nowrap', tdClass: 'align-middle py-3' },
      { key: 'status', sortable: true, thClass: 'text-nowrap', tdClass: 'align-middle py-3' },
      { key: 'driver', label: 'Driver', thClass: 'text-nowrap', tdClass: 'text-nowrap align-middle text-center py-3' },
      { key: 'car', label: 'Car', thClass: 'text-nowrap', tdClass: 'text-nowrap align-middle text-center py-3' },
      { key: 'certificate', label: 'Certificate', thClass: 'text-nowrap', tdClass: 'text-nowrap align-middle text-center py-3' },
      { key: 'seats', label: 'Seats', thClass: 'text-nowrap', tdClass: 'text-nowrap align-middle text-center py-3' },
      { key: 'action', label: 'Action', thClass: 'text-nowrap', tdClass: 'text-nowrap align-middle text-center py-3' }
    ],

    departure_from: 'Any',
    departure_to: 'Any',
    departureDate: null,

    ticketsData: [],
    originalTicketsData: [],
    seats: []
  }),

  computed: {
    totalItems () {
      return this.ticketsData.length
    },
    totalPages () {
      return Math.ceil(this.totalItems / this.perPage)
    },
    currentUser () {
      return this.$store.state.userLoggedIn
    }
  },

  methods: {
    showModal_avatar(driver){
        let url = this.getImage(driver.id, 'avatar');
        let name = driver.firstName + " " + driver.lastName;
        let phone = driver.phone;
        let badges = "";
        let tags = driver.tags;
        let htmlText = '<div>'+phone+'</div>';
        let altMsg = 'Driver \'s avatar';

        if(driver.rating > 0){
          let roundedRating = Math.round( driver.rating * 10 ) / 10;
          htmlText += '<div><font size="2">Rated '+ roundedRating +' on '+driver.review+' reviews</font></div>';
        }

        for(let i in tags){
          badges += '<button class="d-inline-block mr-2" style="border-color: transparent; background: #26B4FF; color: #fff;">' + tags[i] + '</button>';
        }

        swal({
            title: name,
            html: htmlText,
            imageUrl: url,
            imageAlt: altMsg,
            showConfirmButton: false,
            footer: badges,
            buttonsStyling: false,
            animation: false,
            customClass: 'animated zoomIn'
        });
    },

    showModal_car(driver){
        let url = this.getImage(driver.id, 'car');
        let name = "Plate Number: " + driver.car.plateNum;

        swal({
            text: name,
            imageUrl: url,
            // imageWidth: 400,
            // imageHeight: 200,
            imageAlt: 'Driver \'s car',
            showConfirmButton: false,
            animation: false,
            customClass: 'animated zoomIn'
        });
    },

    showModal_certificate(driver){
      let url = this.getImage(driver.id, 'certificate');
      let txt = "Alma mater: " + driver.almaMater;

      swal({
          text: txt,
          imageUrl: url,
          // imageWidth: 400,
          // imageHeight: 200,
          imageAlt: 'Driver \'s certificate',
          showConfirmButton: false,
          animation: false,
          customClass: 'animated zoomIn'
      });
    },

    filter (value) {
      const val = value.toLowerCase()
      const filtered = this.originalTicketsData.filter(d => {
        return Object.keys(d)
          .filter(k => this.searchKeys.includes(k))
          .map(k => String(d[k]))
          .join('|')
          .toLowerCase()
          .indexOf(val) !== -1 || !val
      })
      this.ticketsData = filtered
    },

    reserveSeat(order, i){
      if(this.$store.state.userLoggedIn.currentRole=="Passenger"){
        if(order.status == "Boarding"){
            //change the reserved status in order to change icon
            order.driver.car.seats[i].reserved = !order.driver.car.seats[i].reserved;
            //modify order
            order.driver.car.seats[i].reserved = true;
            order.driver.car.seats[i].passenger = this.$store.state.userLoggedIn;
            order.driver.car.seats[i].rating = 0;

            //post to server to update the order
            let url = this.$store.state.dataUrl+'/orders/update/'+ order.id;
            this.$http.put(url, order).then(response => {
                this.$showNotification('acNotification', 'success', 'Seat-reserve', 'Seat is reserved successfully!');
            }, response => {
                this.$showNotification('acNotification', 'error', 'Seat-reserve', 'Seat is not reserved successfully!');
            });
          }
          else{
              this.$showNotification('acNotification', 'error', 'Seat-reserve', 'Seat in On-board status does not accept reservation!');
          }
        }else{
          this.$showNotification('acNotification', 'error', 'Seat-reserve', 'You do not have right to make this reservation!');
        }
    },

    releaseSeat(order, i){
      if(order.driver.car.seats[i].passenger.id == this.$store.state.userLoggedIn.id && order.status == "Boarding" && this.$store.state.userLoggedIn.currentRole=="Passenger"){
          order.driver.car.seats[i].reserved = !order.driver.car.seats[i].reserved;
          order.driver.car.seats[i].reserved = null;
          order.driver.car.seats[i].passenger = null;
          order.driver.car.seats[i].rating = null;

          //post to server to update the order
          let url = this.$store.state.dataUrl+'/orders/update/'+ order.id;
          this.$http.put(url, order).then(response => {
            this.$showNotification('acNotification', 'success', 'Seat-cancel reserve', 'Seat reservation is cancelled successfully!');
          }, response => {
            this.$showNotification('acNotification', 'error', 'Seat-cancel reserve', 'Seat reservation is not cancelled successfully!');
          });
        }else{
            this.$showNotification('acNotification', 'error', 'Seat-cancel reserve', 'You do not have right to cancel this reservation!');
        }
    },

    getImage(driverId, imageName){
      return this.$store.state.dataUrl + "\\images\\" + driverId + "\\" + imageName + ".jpg";
    },

    startJourney(row){
        row.status = "On-board";
        let url = this.$store.state.dataUrl+'/orders/update/'+ row.id;
        this.$http.put(url, row).then(response => {
            this.$showNotification('acNotification', 'success', 'Ticket-change', 'Have a nice journey!');

            //send message to driver and all passengers
            const msgTypeToDriver = "Notification_Success_Ticket";
            const msgToDriver = "You have changed the order\'s status from Boarding to On-board. Please pick up all passengers on time.";
            const msgTypeToPassenger = "Message";
            const msgToPassenger = "Ticket is authenticated. I will pick you up on time.";
            this.sendMessageToAll(row, msgTypeToDriver, msgToDriver, msgTypeToPassenger, msgToPassenger);

        }, response => {
            this.$showNotification('acNotification', 'error', 'Ticket-change', 'Error occurred when changing ticket\'s status!');
        });
    },

    sendMessageToAll(row, msgTypeToDriver, msgToDriver, msgTypeToPassenger, msgToPassenger){
      const ticketInfo = "Ticket information: from " + row.departureLocation + ", " + row.departureCity + " to " + row.arrivalLocation + ", " + row.arrivalCity+" at " + row.departureTime+" " + row.departureDate;
      let now = new Date().toUTCString();
      let message_driver = {
        type: msgTypeToDriver,
        msgContent: msgToDriver + " " + ticketInfo,
        time: now,
        unread: true
      };
      //send message to the driver
      this.$sendMessage(row.driver, message_driver);

      //send message to all passengers
      let message_passenger = {
        type: msgTypeToPassenger,
        msgContent: msgToPassenger + " " + ticketInfo,
        time: now,
        unread: true,
        sender: row.driver
      };

      console.log("start journey");
      console.log(message_passenger);

      let seats = row.driver.car.seats;
      for(let i=0; i<seats.length; i++){
        let passenger = seats[i].passenger;
        if(passenger != null && passenger.id != null){
          this.$sendMessage(passenger, message_passenger);
        }else{
          continue;
        }
      }
    },

    finishJourney(row){
        if(row.status == "On-board"){
            row.status = "Finished";
            let url = this.$store.state.dataUrl+'/orders/update/'+ row.id;
            this.$http.put(url, row).then(response => {
              //update ticketsData and originalTicketsData
              this.removeOrderFromResult(row);
              this.$showNotification('acNotification', 'success', 'Ticket-change', 'Journey is finished!');

              const msgTypeToDriver = "Notification_Success_Ticket";
              const msgToDriver = "Good job driver. You have finished the journey.";
              const msgTypeToPassenger = "Message";
              const msgToPassenger = "Thank you. Our journey is finished. You can rate me through your finished orders.";
              this.sendMessageToAll(row, msgTypeToDriver, msgToDriver, msgTypeToPassenger, msgToPassenger);

            }, response => {
              this.$showNotification('acNotification', 'error', 'Ticket-change', 'Error occurred when changing ticket\'s status!');
            });
        }else{
            this.$showNotification('acNotification', 'error', 'Ticket-change', 'Boarding order can not be changed directly to finished!');
        }
    },

    deleteTicket(row){
      this.$http.delete(this.$store.state.dataUrl+'/orders/delete/'+row.id).then(response => {
        //update ticketsData and originalTicketsData
        this.removeOrderFromResult(row);
        this.$showNotification('acNotification', 'warn', 'Ticket-delete', 'Ticket is deleted successfully!');

        //send message to driver and all passengers
        const msgTypeToDriver = "Notification_Warn_Ticket";
        const msgToDriver = "You have cancelled the journey. We understand your inconvenience, but please explore more to passengers.";
        const msgTypeToPassenger = "Message";
        const msgToPassenger = "I am sorry that I have cancelled the journey.";
        this.sendMessageToAll(row, msgTypeToDriver, msgToDriver, msgTypeToPassenger, msgToPassenger);

      }, response => {
        this.$showNotification('acNotification', 'error', 'Ticket-delete', 'Ticket is not deleted successfully!');
      });
    },

    removeOrderFromResult(item){
        this.ticketsData.splice(this.ticketsData.indexOf(item), 1);
        this.originalTicketsData.splice(this.originalTicketsData.indexOf(item), 1);
    },

    show(from, to, date){
      let url =  this.$store.state.dataUrl+'/orders/get/allOngoing?departureCity='+ from + '&arrivalCity='+to+'&departureDate='+date;
      const req = new XMLHttpRequest()
      req.open('GET', url)
      req.onload = () => {
        const data = JSON.parse(req.response)
        this.ticketsData = data
        this.originalTicketsData = data.slice(0)
      }
      req.send()
    }
  }
}
</script>
