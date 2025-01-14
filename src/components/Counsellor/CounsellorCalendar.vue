<template>
  <div v-if="user">
    <div v-if="createSession == true">
      <br />
      <p class="text-lg">Select Date to Open Up New Slot</p>
      <br />
      <div id="backBtn">
        <button class="btn btn-link text-info" v-on:click="back()">
          ← Back to Appointments Calendar
        </button>
      </div>

      <v-date-picker v-model="date" mode="dateTime" />
      <br /><br />
      <button class="btn btn-primary btn-sm mb-4" v-on:click="create()">
        Open Up Slot
      </button>
    </div>
    <div v-else>
      <br />
      <p class="text-lg">Select Date to View Upcoming Appointments</p>
      <br />

      <v-date-picker
        v-model="date"
        :attributes="attributes"
        @dayclick="onDayClick"
      />
      <br /><br />
      <div v-if="upcoming.length != 0">
        <p><strong>Upcoming Sessions</strong></p>
        <div v-for="item in upcoming" :key="item">
          {{ item.date }} {{ item.time }}
        </div>
        <br />
      </div>
      <div v-if="avail.length != 0">
        <p><strong>Available Sessions</strong></p>
        <div v-for="item in avail" :key="item">
          {{ item.date }} {{ item.time }}
          <button
            id="cancelBtn"
            class="btn btn-xs btn-outline"
            v-on:click="cancel(this.counsellor_ID, item)"
          >
            Cancel
          </button>
        </div>
        <br />
      </div>
      <div v-if="upcoming.length == 0 && avail.length == 0">
        <h4>No session for selected day</h4>
      </div>
      <br />
      <button
        id="addSessionBtn"
        class="btn btn-sm btn-primary"
        v-on:click="createSession = true"
      >
        Add New Slot
      </button>
    </div>
  </div>
</template>

<script>
import { getAuth, onAuthStateChanged } from "firebase/auth";
import firebaseApp from "../../firebase.js";
import { getFirestore } from "firebase/firestore";
import {
  doc,
  getDoc,
  setDoc,
  updateDoc,
  arrayRemove,
  deleteDoc,
} from "firebase/firestore";
const db = getFirestore(firebaseApp);

export default {
  name: "CounsellorCalendar",
  data() {
    return {
      user: false,
      user_type: "patient",
      counsellor_ID: this.$route.params.id, // UID
      display_avail: [],
      display_upcoming: [],
      upcoming: [],
      avail: [],
      date: new Date(),
      attributes: [],
      createSession: false,
    };
  },
  mounted() {
    const auth = getAuth();
    onAuthStateChanged(auth, (user) => {
      if (user) {
        this.user = user;
        if (user.user_type == "counsellor") {
          this.user_type = "counsellor";
        }
        console.log("bottom of mounted()");
      }
    });
    this.getAttributes();
  },
  methods: {
    async getAttributes() {
      const docRef = doc(db, "Counsellors", this.counsellor_ID);
      const docSnap = await getDoc(docRef);
      let z = docSnap.data();
      //counsellor available sessions
      z.available_slots.forEach((a) => {
        if (a.toDate() > new Date()) {
          this.display_avail.push(a.toDate());
        }
      });
      //counsellor upcoming sessions
      for (const u of z.upcoming_counsellor_sessions) {
        const sessionSnap = await getDoc(doc(db, "Sessions", u));
        const slotTime = sessionSnap.data().session_time; // this is in local time.
        if (slotTime.toDate() > new Date()) {
          this.display_upcoming.push(slotTime.toDate());
        }
      }

      //green bar for avail, red bar for upcoming
      this.attributes = [
        {
          bar: "green",
          dates: this.display_avail,
        },
        {
          bar: "red",
          dates: this.display_upcoming,
        },
      ];
    },
    async onDayClick(day) {
      const docRef = doc(db, "Counsellors", this.counsellor_ID);
      const docSnap = await getDoc(docRef);
      let z = docSnap.data();
      let avail = z.available_slots;
      let upcoming = z.upcoming_counsellor_sessions;

      this.avail.splice(0, avail.length);
      this.upcoming.splice(0, upcoming.length);

      upcoming.forEach(async (s) => {
        const sessionRef = doc(db, "Sessions", s);
        const sessionSnap = await getDoc(sessionRef);
        let session = sessionSnap.data().session_time;

        let a = session.toDate();
        if (a > new Date()) {
          if (a.toDateString() == day.date.toDateString()) {
            this.upcoming.push({
              date: a.toDateString(),
              time: a.toTimeString().substr(0, 5),
            });
          }

          this.upcoming.sort((x, y) => {
            return x.session - y.session;
          });
        }
      });

      avail.forEach(async (x) => {
        const slotRef = doc(
          db,
          "Sessions",
          this.counsellor_ID + x.toDate().toUTCString()
        );
        const slotSnap = await getDoc(slotRef);
        let slot = slotSnap.data().session_time;
        let s = slot.toDate();

        console.log("s is ", s);
        console.log("s datestring is", s.toDateString());
        console.log("day.date is", day.date.toDateString());

        if (s > new Date()) {
          if (s.toDateString() == day.date.toDateString()) {
            this.avail.push({
              date: s.toDateString(),
              time: s.toTimeString().substr(0, 5),
              session: s,
            });
          }

          this.avail.sort((x, y) => {
            return x.session - y.session;
          });
        }
      });
    },
    async create() {
      //edit counsellor > add to available_slots
      const docRef = doc(db, "Counsellors", this.counsellor_ID);
      const docSnap = await getDoc(docRef);
      let z = docSnap.data();
      let avail = z.available_slots;
      let upcoming = z.upcoming_counsellor_sessions;

      //check for time clash
      let one_hour = 60 * 60 * 1000;
      let same_time_session = null;
      upcoming.forEach((u) => {
        if (Math.abs(new Date(u.substring(28)) - this.date) < one_hour) {
          same_time_session = u.substring(28, 49);
        }
      });
      avail.forEach((a) => {
        if (Math.abs(a.toDate() - this.date) < one_hour) {
          same_time_session = a
            .toDate()
            .toString()
            .substring(0, 21);
        }
      });

      if (this.date < new Date()) {
        // past date
        alert("Session cannot be created in the past");
      } else if (same_time_session) {
        // time clash
        alert(
          "There is a clash with an existing session on " +
            same_time_session +
            ". Please do not add a new slot within an hour of an existing available slot or upcoming appointment."
        );
      } else {
        avail.push(this.date);
        await setDoc(docRef, { available_slots: avail }, { merge: true });

        //create new session
        await setDoc(
          doc(db, "Sessions", this.counsellor_ID + this.date.toUTCString()),
          {
            counsellor_ID: this.counsellor_ID,
            rating: null,
            room_ID: "",
            session_notes: "",
            session_time: this.date,
            user_ID: "",
          }
        );
        alert("New session created for " + this.date + " !");
      }
    },
    async back() {
      this.createSession = false;
      location.reload();
    },
    async cancel(counsellor, item) {
      var confirmCancel = confirm(
        "Press 'OK' to proceed to cancel this appointment on " +
          item.date +
          " " +
          item.time
      );

      if (confirmCancel) {
        //pressed OK
        //remove from counsellor available slots
        const counsellorRef = doc(db, "Counsellors", this.counsellor_ID);
        await updateDoc(counsellorRef, {
          available_slots: arrayRemove(item.session),
        });

        //delete from sessions
        await deleteDoc(
          doc(db, "Sessions", this.counsellor_ID + item.session.toUTCString())
        );

        alert(
          "Session on " + item.date + " " + item.time + " has been cancelled"
        );
        location.reload();
      }
    },
  },
};
</script>

<style scoped>
#addSessionBtn {
  margin-left: 5px;
  margin-right: 5px;
  margin-bottom: 20px;
}

#cancelBtn {
  margin-bottom: 5px;
  margin-left: 5px;
}
</style>
