<template>
  <div>
    <div class="container">
      <h1 class="text-center">Loan State Machine</h1>

      <p>Borrower: {{ loan.borrower }}</p>
      <p>Lender: {{ loan.lender }}</p>
      <p>Amount: {{ loan.amount }} Wei</p>
      <p>Interest: {{ loan.interest }} Wei</p>
      <p>DurationI: {{ loan.duration }}</p>

      <p v-if="loan.state > 0">
        End: {{ new Date(parseInt(loan.end) * 1000).toLocaleString() }}
      </p>

      <p>State: {{ states[loan.state] }}</p>

      <div class="row" v-if="activeAccount === loan.lender && loan.state === 0">
        <div class="col-sm-12">
          <h2>Fund {{ loan.amount }} Wei</h2>
          <button @click.prevent="fund" type="submit" class="btn btn-primary">
            Submit
          </button>
        </div>
      </div>

      <div
        class="row"
        v-if="
          activeAccount === loan.borrower && loan.state === 1 && isFinished()
        "
      >
        <div class="col-sm-12">
          <h2>
            Reimburse {{ loan.amount }} Wei (principal) + {{ loan.interest }}
            (interest)
          </h2>
          <button
            @click.prevent="reimburse"
            type="submit"
            class="btn btn-primary"
          >
            Submit
          </button>
        </div>
      </div>
    </div>
  </div>
</template>
<script>
import { mapGetters } from "vuex";
export default {
  data: () => ({
    loan: undefined,
    states: ["PENDING", "ACTIVE", "CLOSED"],
  }),
  computed: {
    ...mapGetters("accounts", ["activeAccount"]),
    ...mapGetters("drizzle", ["drizzleInstance"]),
  },
  methods: {
    fund() {
      let amount = this.loan.amount;
      this.drizzleInstance.contracts.StateMachine.methods
        .fund()
        .send({
          from: this.activeAccount,
          value: amount,
        })
        .then(() => {
          this.init();
        });
    },
    reimburse() {
      let amount = parseInt(this.loan.amount);
      let interest = parseInt(this.loan.interest);
      let sum = amount + interest;
      this.drizzleInstance.contracts.StateMachine.methods
        .reimburse()
        .send({
          from: this.activeAccount,
          value: sum,
        })
        .then(() => {
          this.init();
        });
    },
    isFinished() {
      const now = new Date().getTime();
      const loanEnd = new Date(parseInt(this.loan.end) * 1000).getTime();
      return loanEnd > now ? false : true;
    },
    async init() {
      const contract = this.drizzleInstance.contracts.StateMachine;
      const res = await Promise.all([
        contract.methods.amount().call(),
        contract.methods.interest().call(),
        contract.methods.end().call(),
        contract.methods.duration().call(),
        contract.methods.borrower().call(),
        contract.methods.lender().call(),
        contract.methods.state().call(),
      ]);

      let loan = {};
      loan.amount = res[0];
      loan.interest = res[1];
      loan.end = res[2];
      loan.duration = res[3];
      loan.borrower = res[4];
      loan.lender = res[5];
      loan.state = parseInt(res[6]);
      this.loan = loan;
    },
  },
  created() {
    this.init();
  },
};
</script>
