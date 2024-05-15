<script setup lang="ts">
import { ref, reactive, computed } from "vue";

import { Line } from "vue-chartjs";
import {
    Chart as ChartJS,
    Title,
    Tooltip,
    Legend,
    PointElement,
    LineElement,
    CategoryScale,
    LinearScale,
    Utils,
} from "chart.js";

ChartJS.register(
    Title,
    Tooltip,
    Legend,
    PointElement,
    LineElement,
    CategoryScale,
    LinearScale,
);
ChartJS.defaults.color = "white";

const chartOptions = {
    scales: {
        y: {
            title: {
                display: true,
                text: "Money ($)",
            },
        },
        x: {
            title: {
                display: true,
                text: "Month #",
            },
        },
    },
};
// ChartJS.defaults.global.defaultFontColor = "white";

const mortgageParams = reactive({
    principal: 1015000,
    interest: 3.625,
    numYears: 30,
});

const personalLoanParams = reactive({
    principal: 454741.93,
    interest: 5,
    numYears: 30,
    inflationRate: 2,
    inflateWithMortgagePayment: true,
});

const houseParams = reactive({
    numPeople: 7,
    utilities: 2612.62,
    utilityInflation: 2,
});

function monthlyLoanPayment(principal, interest, numYears) {
    if (interest <= 0) {
        return principal / (numYears * 12);
    } else {
        const monthlyInterestRate = interest / 100 / 12;
        const numberOfPayments = numYears * 12;
        return (
            (principal * monthlyInterestRate) /
            (1 - Math.pow(1 + monthlyInterestRate, -numberOfPayments))
        );
    }
}

const monthlyMortgagePayment = computed(() => {
    const { principal, interest, numYears } = mortgageParams;
    return monthlyLoanPayment(principal, interest, numYears);
});

function oneMonthBalanceAndPaymentUpdate(
    balance: number,
    monthlyPayment: number,
    month: number,
) {
    balance *= 1 + personalLoanParams.interest / 100 / 12;

    if (month && month % 12 === 0) {
        if (personalLoanParams.inflateWithMortgagePayment) {
            monthlyPayment += monthlyMortgagePayment.value;
        }
        monthlyPayment *= 1 + personalLoanParams.inflationRate / 100;
        if (personalLoanParams.inflateWithMortgagePayment) {
            monthlyPayment -= monthlyMortgagePayment.value;
        }
    }

    balance -= monthlyPayment;

    return [balance, monthlyPayment];
}

const startingMonthlyPersonalLoanPayment = computed(() => {
    const { principal, interest, numYears } = personalLoanParams;

    // Do binary search for the starting monthly payment
    let startingMonthlyPaymentMid = monthlyLoanPayment(
        principal,
        interest,
        numYears,
    );
    let startingMonthlyPaymentLow = 0;
    let startingMonthlyPaymentHigh = null; // null means unbounded

    for (let j = 0; j < 100; j++) {
        let monthlyPayment = startingMonthlyPaymentMid;
        let balance = principal;
        for (let i = 0; i < numYears * 12; i++) {
            [balance, monthlyPayment] = oneMonthBalanceAndPaymentUpdate(
                balance,
                monthlyPayment,
                i,
            );
        }

        if (Math.abs(balance) < 0.01) {
            return startingMonthlyPaymentMid;
        }

        if (balance > 0) {
            if (startingMonthlyPaymentHigh === null) {
                startingMonthlyPaymentLow = startingMonthlyPaymentMid;
                startingMonthlyPaymentMid *= 2;
            } else {
                startingMonthlyPaymentLow = startingMonthlyPaymentMid;
                startingMonthlyPaymentMid =
                    (startingMonthlyPaymentMid + startingMonthlyPaymentHigh) /
                    2;
            }
        } else {
            startingMonthlyPaymentHigh = startingMonthlyPaymentMid;
            startingMonthlyPaymentMid =
                (startingMonthlyPaymentMid + startingMonthlyPaymentLow) / 2;
        }
    }
    return NaN;
});

function loanToIndividualMonthlyPayment(personalLoanMonthlyPayment, month) {
    const { numPeople, utilities, utilityInflation } = houseParams;

    // Inflate utilies once a year
    const year = Math.floor(month / 12);
    const utilitiesNow = utilities * Math.pow(1 + utilityInflation / 100, year);

    const totalMonthlyPayment =
        personalLoanMonthlyPayment +
        utilitiesNow +
        monthlyMortgagePayment.value;
    return totalMonthlyPayment / numPeople;
}

const chronologicalData = computed(() => {
    const { principal, numYears } = personalLoanParams;

    const personalLoanBalance = [];
    const personalLoanMonthlyPayment = [];
    const individualMonthlyPayment = [];
    let totalLoanPayment = 0.0;

    let balance = principal;
    let monthlyPayment = startingMonthlyPersonalLoanPayment.value;
    for (let i = 0; i < numYears * 12; i++) {
        [balance, monthlyPayment] = oneMonthBalanceAndPaymentUpdate(
            balance,
            monthlyPayment,
            i,
        );
        personalLoanBalance.push(balance);
        personalLoanMonthlyPayment.push(monthlyPayment);
        individualMonthlyPayment.push(
            loanToIndividualMonthlyPayment(monthlyPayment, i),
        );
        totalLoanPayment += monthlyPayment;
    }
    return {
        personalLoanBalance,
        personalLoanMonthlyPayment,
        individualMonthlyPayment,
        totalLoanPayment,
    };
});

const totalLoanPayment = computed(() => {
    return chronologicalData.value.totalLoanPayment;
});

const chartData = computed(() => {
    const data = chronologicalData.value;
    return {
        labels: Array.from(
            { length: data.personalLoanBalance.length },
            (_, i) => i,
        ),
        datasets: [
            {
                label: "Individual Monthly Payment (including utilities)",
                backgroundColor: "purple",
                data: data.individualMonthlyPayment,
            },
            {
                label: "Personal Loan Monthly Payment (total)",
                backgroundColor: "skyblue",
                data: data.personalLoanMonthlyPayment,
            },
            {
                label: "Personal Loan Balance",
                backgroundColor: "salmon",
                data: data.personalLoanBalance,
                hidden: true,
            },
        ],
    };
});
</script>

<template>
    <h1>Horse House Finance Calculator</h1>
    <fieldset>
        <legend>Mortgage</legend>
        <details>
            <ul>
                <li>
                    <label for="mortgagePrincipal">Principal Money Owed</label>
                    <input
                        id="mortgagePrincipal"
                        type="number"
                        v-model="mortgageParams.principal"
                        step="10000"
                    />
                </li>
                <li>
                    <label for="mortgageInterest">Yearly Interest</label>
                    <input
                        id="mortgageInterest"
                        type="number"
                        v-model="mortgageParams.interest"
                        step="0.1"
                    />
                </li>
                <li>
                    <label for="mortgageYears">Number of Years</label>
                    <input
                        id="mortgageYears"
                        type="number"
                        v-model="mortgageParams.numYears"
                    />
                </li>
            </ul>
            <summary>
                Mortgage Monthly Payment:
                <strong> ${{ monthlyMortgagePayment.toFixed(2) }} </strong>
                <em> (click to expand) </em>
            </summary>
        </details>
    </fieldset>

    <fieldset>
        <legend>Personal Loan</legend>
        <details>
            <ul>
                <li>
                    <label for="loanPrincipal"
                        >Principal Money Owed (i.e. Downpayment & Closing Costs)
                    </label>
                    <input
                        id="loanPrincipal"
                        type="number"
                        v-model="personalLoanParams.principal"
                        step="10000"
                    />
                </li>
                <li>
                    <label for="totalLoan"
                        >Total Loan Payed (including interest and inflation)
                    </label>
                    <text> ${{ totalLoanPayment.toFixed(2) }} </text>
                </li>
                <li>
                    <label for="loanInterest">Yearly Interest</label>
                    <input
                        id="loanInterest"
                        type="number"
                        v-model="personalLoanParams.interest"
                        step="0.1"
                    />
                </li>
                <li>
                    <label for="loanYears">Number of Years</label>
                    <input
                        id="loanYears"
                        type="number"
                        v-model="personalLoanParams.numYears"
                    />
                </li>
                <li>
                    <label for="inflationRate">Yearly Payment Inflation</label>
                    <input
                        id="inflationRate"
                        type="number"
                        v-model="personalLoanParams.inflationRate"
                        step="0.1"
                    />
                </li>
                <li>
                    <label for="inflateWithMortgagePayment"
                        >Inflate with Mortgage Payment - this inflates the
                        personal loan payment such that it <em>plus</em> the
                        (fixed) mortgage payment grows at the inflation
                        rate</label
                    >
                    <input
                        id="inflateWithMortgagePayment"
                        type="checkbox"
                        v-model="personalLoanParams.inflateWithMortgagePayment"
                    />
                </li>
            </ul>
            <summary>
                Initial Personal Loan Monthly Payment:
                <strong
                    >${{
                        startingMonthlyPersonalLoanPayment.toFixed(2)
                    }}</strong
                >
                <em> (click to expand) </em>
            </summary>
        </details>
    </fieldset>

    <fieldset>
        <legend>Non-Loan Factors</legend>
        <details>
            <ul>
                <li>
                    <label for="numPeople">Number of People</label>
                    <input
                        id="numPeople"
                        type="number"
                        v-model="houseParams.numPeople"
                    />
                </li>
                <li>
                    <label for="utilities"
                        >Total Monthly Utilities (including gas, electric,
                        water, sewer, internet, insurance, taxes, house project
                        fund)</label
                    >
                    <input
                        id="utilities"
                        type="number"
                        v-model="houseParams.utilities"
                        step="100"
                    />
                </li>
                <li>
                    <label for="utilityInflation"
                        >Yearly Utility Inflation (estimated)</label
                    >
                    <input
                        id="utilityInflation"
                        type="number"
                        step="0.1"
                        v-model="houseParams.utilityInflation"
                    />
                </li>
            </ul>

            <summary>
                Initial Individual Monthly Payment:
                <strong
                    >${{
                        loanToIndividualMonthlyPayment(
                            startingMonthlyPersonalLoanPayment,
                            0,
                        ).toFixed(2)
                    }}</strong
                >
                <em> (click to expand) </em>
            </summary>
        </details>
    </fieldset>

    <Line :data="chartData" :options="chartOptions" />
</template>
