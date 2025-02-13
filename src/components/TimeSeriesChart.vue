<template>
  <v-card>
    <v-card-actions class="justify-space-between flex-wrap gap-1 py-5 px-5">
      <div class="d-flex align-center flex-wrap gap-1">
        <v-btn-toggle
          dark
          v-model="activeBtn"
          class="mr-5"
        >
          <v-btn
            v-for="serie, i in series"
            :key="i"
            small
            @click="$emit('set-serie', serie)"
          >
            {{ serie | capitalize}}
          </v-btn>
        </v-btn-toggle>

        <v-switch
          v-model="showSMA"
          inset
          :label="`${showSMA ? 'Hide' : 'Show'} SMA`"
          hide-details
          class="my-0 py-0"
        ></v-switch>
      </div>

      <div class="subtitle-1">
        {{ symbol["4. region"] }}
        <v-icon>mdi-circle-small</v-icon>
        {{ symbol["8. currency"] }}
        <v-icon>mdi-circle-small</v-icon>
        {{ symbol["5. marketOpen"] }} - 
        {{ symbol["6. marketClose"] }}
      </div>
    </v-card-actions>
    <v-divider></v-divider>
    <v-card-text>

      <div id="chart"></div>

      <v-overlay
        absolute
        :value="$store.state.isLoading"
      >
        <v-progress-circular
          indeterminate
          size="64"
        ></v-progress-circular>
      </v-overlay>
    </v-card-text>
  </v-card>
</template>

<script>
import * as d3 from "d3";
import { mapGetters } from "vuex";

export default {
  name: "TimeSeriesChart",
  props: {
    draw: Boolean, // chart drawing trigger
    serie: String,  // serie info
    symbol: Object  // symbol info
  },
  data() {
    return {
      showSMA: false, // show simple moving average line in chart

      series: ["daily", "weekly", "monthly"],
      activeBtn: 0, // active serie selection btn
    };
  },
  computed: {
    ...mapGetters(["getFormattedTimeSeries"]),
  },
  watch: {
    // wathes the draw trigger and draws the chart acoordingly
    draw(isDrawing) {
      if (isDrawing) {
        this.drawChart(this.getFormattedTimeSeries(this.serie));
      }
    },
    // whatches the sma trigger and shows the sma graph in chart
    showSMA() {
      this.drawChart(this.getFormattedTimeSeries(this.serie))
    }
  },
  created() {
    // set active btn according to serie
    if(this.serie) {
      this.activeBtn = this.series.indexOf(this.serie)
    }
  },
  methods: {
    // draws the chart
    drawChart(data) {
      d3.selectAll("div#chart > *").remove();

      const width = 1300;
      const height = 500;
      const margin = 50;

      function min(a, b) {
        return a < b ? a : b;
      }

      function max(a, b) {
        return a > b ? a : b;
      }

      const y = d3.scaleLinear().range([height - margin, margin]);

      const x = d3.scaleTime().range([margin, width - margin]);

      // line for the sma
      const line1 = d3
        .line()
        .x(function (d) {
          return x(d["date"]);
        })
        .y(function (d) {
          return y(d["close"]);
        });

      data.forEach(function (d) {
        d.date = new Date(d.date);
        d.high = +d.high;
        d.low = +d.low;
        d.open = +d.open;
        d.close = +d.close;
        d.volume = +d.volume;
      });

      const chart = d3
        .select("div#chart")
        .append("svg")
        .attr("class", "chart")
        .attr("width", width)
        .attr("height", height)
        .attr("viewBox", [0, 0, width, height])
        .attr("style", "max-width: 100%; height: auto; height: intrinsic;");

      y.domain([
        d3.min(
          data.map(function (x) {
            return x["low"];
          })
        ),
        d3.max(
          data.map(function (x) {
            return x["high"];
          })
        ),
      ]);

      x.domain(
        d3.extent(data, function (d) {
          return d["date"];
        })
      );

      chart
        .selectAll("line.x")
        .data(x.ticks(10))
        .enter()
        .append("line")
        .attr("class", "x")
        //.text(String)
        .attr("x1", x)
        .attr("x2", x)
        .attr("y1", margin)
        .attr("y2", height - margin)
        .attr("stroke", "#ccc");
      chart
        .selectAll("line.y")
        .data(y.ticks(10))
        .enter()
        .append("line")
        .attr("class", "y")
        .attr("x1", margin)
        .attr("x2", width - margin)
        .attr("y1", y)
        .attr("y2", y)
        .attr("stroke", "#ccc");

      chart
        .append("g")
        .attr("transform", "translate(0," + 450 + ")") //need to change this 450 to a variable- it is how far down the axis will go
        .attr("class", "xrule")
        .call(d3.axisBottom(x))
        .selectAll("text")
        .style("text-anchor", "end")
        .attr("dx", "-.8em")
        .attr("dy", ".15em")
        .attr("transform", function () {
          return "rotate(-65)";
        });

      chart
        .selectAll("text.yrule")
        .data(y.ticks(10))
        .enter()
        .append("text")
        .attr("class", "yrule")
        .attr("x", 0)
        .attr("y", y)
        .attr("dy", 0)
        .attr("dx", 20)
        .attr("text-anchor", "middle")
        .text(String);

      chart
        .selectAll("rect")
        .data(data)
        .enter()
        .append("rect")
        .attr("x", function (d) {
          return x(d["date"]);
        })
        .attr("y", function (d) {
          return y(max(d["open"], d["close"]));
        })
        .attr("height", function (d) {
          return y(min(d["open"], d["close"])) - y(max(d["open"], d["close"]));
        })
        .attr("width", function () {
          return (0.6 * (width - 2 * margin)) / data.length;
        })
        .attr("fill", function (d) {
          return d["open"] > d["close"] ? "red" : "green";
        });

      chart
        .selectAll("line.stem")
        .data(data)
        .enter()
        .append("line")
        .attr("class", "stem")
        .attr("x1", function (d) {
          return x(d["date"]) + (0.25 * (width - 2 * margin)) / data.length;
        })
        .attr("x2", function (d) {
          return x(d["date"]) + (0.25 * (width - 2 * margin)) / data.length;
        })
        .attr("y1", function (d) {
          return y(d["high"]);
        })
        .attr("y2", function (d) {
          return y(d["low"]);
        })
        .attr("stroke", function (d) {
          return d.open > d.close ? "red" : "green";
        });

      // sma
      chart
        .append("path")
        .data([data])
        .attr("d", line1)
        .attr("class", "line")
        .style("stroke", "red")
        .attr("fill", "none")
        .attr("stroke-width", 1)
        .classed("hide-sma", !this.showSMA)
    },
  },
};
</script>

<style>
.hide-sma {
  display: none;
}
</style>