<script>
(function () {
  // =========================
  // Elements
  // =========================
  const form = document.getElementById("ggCalcForm");
  if (!form) return;

  const stepInputs = document.getElementById("ggCalcStepInputs");
  const stepOutputs = document.getElementById("ggCalcStepOutputs");
  const outerWrap = document.getElementById("ggCalcOuterWrap");
  const chartPlaceholder = document.getElementById("ggChartPlaceholder");
  const chartReal = document.getElementById("ggChartReal");
  const disclaimerEl = document.getElementById("ggCalculatorDisclaimer");
  const err = document.getElementById("ggCalcError");

  const weeklyApptsEl = document.getElementById("weeklyAppts");
  const staffCountEl  = document.getElementById("staffCount");
  const avgTicketEl   = document.getElementById("avgTicket");
  const pctComeBackEl = document.getElementById("pctComeBack");

  const revBeforeEl = document.getElementById("ggRevBefore");
  const revAfterEl  = document.getElementById("ggRevAfter");

  const leakReturningEl  = document.getElementById("ggLeakReturning");
  const leakBookingsEl   = document.getElementById("ggLeakBookings");
  const leakTicketEl     = document.getElementById("ggLeakTicket");
  const leakProcessingEl = document.getElementById("ggLeakProcessing");

  const backBtn  = document.getElementById("ggCalcBack");
  const resetBtn = document.getElementById("ggCalcReset");

  // ✅ Segment: submit button with ID calcSubmit (kept, but NOT used for tracking click)
  const submitBtn = document.getElementById("calcSubmit");

  // ✅ CTA buttons (hide by default)
  const freeTrialBtn = document.getElementById("ggCalcFreeTrialBtn");
  const demoBtn = document.getElementById("ggCalcDemoBtn");

  // Hide by default
  if (freeTrialBtn) freeTrialBtn.style.display = "none";
  if (demoBtn) demoBtn.style.display = "none";
  if (disclaimerEl) disclaimerEl.style.display = "none";

  function setMoreRevText(text) {
    document.querySelectorAll("#ggMoreRev").forEach(n => n.textContent = text);
  }

  // =========================
  // Segment helpers
  // =========================
  const SEGMENT_PROPS = {object_name: "growth_calculator" };

  function segmentTrack(eventName, props) {
    try {
      if (window.analytics && typeof window.analytics.track === "function") {
        window.analytics.track(eventName, props || {});
      }
    } catch (e) {}
  }

  let ggInteractedFired = false;
  function fireInteractedOnce() {
    if (ggInteractedFired) return;
    ggInteractedFired = true;
    segmentTrack("User Interacted with Custom Module", SEGMENT_PROPS);
  }

  [weeklyApptsEl, staffCountEl, avgTicketEl, pctComeBackEl].forEach((el) => {
    if (!el) return;
    el.addEventListener("focus", fireInteractedOnce, true);
    el.addEventListener("click", fireInteractedOnce, true);
  });

  // ✅ IMPORTANT CHANGE:
  // Removed submitBtn click tracking (it fired even when validation fails).

  // =========================
  // Constants (Model B)
  // =========================
  const CONST = {
    apptLift: 0.22,
    ticketLift: 0.30,
    revLift: 0.35,
    rebookFloor: 0.75,
    procGG: 0.026,
    procAvg: 0.035
  };

  const MONTHS = ["Jan","Feb","Mar","Apr","May","Jun","Jul","Aug","Sep","Oct","Nov","Dec"];
  const WEIGHTS = [0.06,0.06,0.08,0.08,0.09,0.09,0.08,0.06,0.08,0.09,0.11,0.12];

  const fmtUsd = (n) =>
    new Intl.NumberFormat("en-US", {
      style: "currency",
      currency: "USD",
      maximumFractionDigits: 0
    }).format(Math.round(n));

  // =========================
  // Inputs
  // =========================
  function getInputs() {
    const weeklyApptsPerStaff = Number(weeklyApptsEl?.value);
    const staffCount = Number(staffCountEl?.value);
    const avgTicket = Number(avgTicketEl?.value);
    const rebookRate = Number(pctComeBackEl?.value) / 100;

    const valid =
      weeklyApptsPerStaff > 0 &&
      staffCount >= 1 &&
      avgTicket >= 0 &&
      rebookRate >= 0 &&
      rebookRate <= 1;

    return { weeklyApptsPerStaff, staffCount, avgTicket, rebookRate, valid };
  }

  // =========================
  // Model B Compute
  // =========================
  function computeModelB({ weeklyApptsPerStaff, staffCount, avgTicket, rebookRate }) {
    const apptVolCurrent = weeklyApptsPerStaff * staffCount * 52;
    const apptVolNew = apptVolCurrent * (1 + CONST.apptLift);

    const currentRevenue = apptVolCurrent * avgTicket;
    const revPotential = currentRevenue * CONST.revLift;
    const procSaving = (CONST.procAvg - CONST.procGG) * currentRevenue;

    const rebookCurrent = Math.max(0, Math.min(1, rebookRate));
    const rebookNew =
      rebookCurrent < CONST.rebookFloor
        ? CONST.rebookFloor
        : Math.min(1, rebookCurrent * 1.1);

    const revReturningCurrent = rebookCurrent * apptVolCurrent * avgTicket;
    const revReturningNew = rebookNew * apptVolNew * avgTicket;

    const ticketRevNew = currentRevenue * (1 + CONST.ticketLift);
    const totalRevNew = apptVolNew * avgTicket;

    const deltaBookingsRaw = totalRevNew - currentRevenue;
    const deltaReturningRaw = revReturningNew - revReturningCurrent;
    const deltaTicketRaw = ticketRevNew - currentRevenue;

    const weightsTotal = deltaBookingsRaw + deltaReturningRaw + deltaTicketRaw || 1;

    const leakBookings = revPotential * (deltaBookingsRaw / weightsTotal);
    const leakReturning = revPotential * (deltaReturningRaw / weightsTotal);
    const leakTicket = revPotential * (deltaTicketRaw / weightsTotal);

    const moreMaking = revPotential + procSaving;
    const projectedRevenue = currentRevenue + moreMaking;

    return {
      currentRevenue,
      projectedRevenue,
      moreMaking,
      leakBookings,
      leakReturning,
      leakTicket,
      procSaving
    };
  }

  // =========================
  // Chart
  // =========================
  function renderRevenueChart(before, after) {
    if (typeof Chart === "undefined") return;

    const canvas = document.getElementById("ggRevChart");
    if (!canvas) return;

    const current = WEIGHTS.map((w) => Math.round(before * w));
    const projected = WEIGHTS.map((w) => Math.round(after * w));

    const BRAND = {
      primary: "#b3bae8",
      bg: "#ffffff",
      tick: "rgba(38,48,46,0.55)",
      tooltipBg: "#26302E",
      currentLine: "rgba(38,48,46,0.25)",
      currentFillTop: "rgba(38,48,46,0.09)",
      currentFillBottom: "rgba(38,48,46,0.02)",
    };

    const ctx = canvas.getContext("2d");
    if (!ctx) return;

    const h = canvas.parentElement?.getBoundingClientRect().height || 160;

    const currentGrad = ctx.createLinearGradient(0, 0, 0, h);
    currentGrad.addColorStop(0, BRAND.currentFillTop);
    currentGrad.addColorStop(1, BRAND.currentFillBottom);

    const potentialGrad = ctx.createLinearGradient(0, 0, 0, h);
    potentialGrad.addColorStop(0, "rgba(179,186,232,0.55)");
    potentialGrad.addColorStop(1, "rgba(179,186,232,0.10)");

    if (window.ggChart) window.ggChart.destroy();

    window.ggChart = new Chart(ctx, {
      type: "line",
      data: {
        labels: MONTHS,
        datasets: [
          {
            label: "Current",
            data: current,
            borderColor: BRAND.currentLine,
            backgroundColor: currentGrad,
            fill: true,
            tension: 0.22,
            borderWidth: 3,
            pointRadius: 0,
            pointHoverRadius: 5,
            pointHoverBorderWidth: 2,
            pointHitRadius: 18,
            pointHoverBackgroundColor: BRAND.bg,
            pointHoverBorderColor: BRAND.currentLine,
          },
          {
            label: "Potential",
            data: projected,
            borderColor: BRAND.primary,
            backgroundColor: potentialGrad,
            fill: "-1",
            tension: 0.22,
            borderWidth: 4,
            borderCapStyle: "round",
            borderJoinStyle: "round",
            pointRadius: 0,
            pointHoverRadius: 6,
            pointHoverBorderWidth: 2,
            pointHitRadius: 18,
            pointHoverBackgroundColor: BRAND.bg,
            pointHoverBorderColor: BRAND.primary,
          },
        ],
      },
      options: {
        responsive: true,
        maintainAspectRatio: false,
        layout: { padding: { top: 6, left: 0, right: 0, bottom: 18 } },
        interaction: { mode: "index", intersect: false },
        plugins: {
          legend: { display: false },
          tooltip: {
            backgroundColor: BRAND.tooltipBg,
            padding: 10,
            displayColors: false,
            caretSize: 0,
            cornerRadius: 10,
            callbacks: {
              title: (items) => items[0].label,
              label: (item) => {
                const val = Math.round(item.parsed.y);
                const i = item.dataIndex;
                const cur = current[i];
                const delta = val - cur;

                if (item.datasetIndex === 1) {
                  return [
                    `Current: $${cur.toLocaleString()}/mo`,
                    `Potential: $${val.toLocaleString()}/mo`,
                    `+${delta.toLocaleString()} vs current`,
                  ];
                }
                return `Current: $${val.toLocaleString()}/mo`;
              },
            },
          },
          datalabels: { display: false },
        },
        scales: {
          x: {
            grid: { display: false },
            border: { display: false },
            ticks: {
              color: BRAND.tick,
              font: { size: 12, weight: "500" },
              padding: 14,
              callback: (val, index) => MONTHS[index],
            },
          },
          y: {
            display: false,
            grid: { display: false },
            border: { display: false },
          },
        },
        elements: { line: { capBezierPoints: false } },
      },
    });
  }

  // =========================
  // UI State
  // =========================
  function showOutputs(model) {
    if (revBeforeEl) revBeforeEl.textContent = fmtUsd(model.currentRevenue);
    if (revAfterEl)  revAfterEl.textContent  = fmtUsd(model.projectedRevenue);
    setMoreRevText(fmtUsd(model.moreMaking));

    if (leakReturningEl)  leakReturningEl.textContent  = fmtUsd(model.leakReturning) + "/yr";
    if (leakBookingsEl)   leakBookingsEl.textContent   = fmtUsd(model.leakBookings) + "/yr";
    if (leakTicketEl)     leakTicketEl.textContent     = fmtUsd(model.leakTicket) + "/yr";
    if (leakProcessingEl) leakProcessingEl.textContent = fmtUsd(model.procSaving) + "/yr";

    if (stepInputs) stepInputs.style.display = "none";
    if (stepOutputs) stepOutputs.style.display = "block";
    if (outerWrap) outerWrap.style.display = "none";
    if (disclaimerEl) disclaimerEl.style.display = "block";

    if (chartPlaceholder) chartPlaceholder.style.display = "none";
    if (chartReal) chartReal.style.display = "block";

    // ✅ CTA logic (<=5 free trial, >5 demo)
    const staffCount = Number(staffCountEl?.value);

    if (freeTrialBtn) freeTrialBtn.style.display = "none";
    if (demoBtn) demoBtn.style.display = "none";

    if (!Number.isNaN(staffCount)) {
      if (staffCount <= 5) {
        if (freeTrialBtn) freeTrialBtn.style.display = "inline-flex";
      } else {
        if (demoBtn) demoBtn.style.display = "inline-flex";
      }
    }

    requestAnimationFrame(() => renderRevenueChart(model.currentRevenue, model.projectedRevenue));
  }

  function showInputs() {
    if (stepOutputs) stepOutputs.style.display = "none";
    if (stepInputs) stepInputs.style.display = "grid";
    if (outerWrap) outerWrap.style.display = "block";
    if (disclaimerEl) disclaimerEl.style.display = "none";

    if (chartPlaceholder) chartPlaceholder.style.display = "block";
    if (chartReal) chartReal.style.display = "none";

    if (freeTrialBtn) freeTrialBtn.style.display = "none";
    if (demoBtn) demoBtn.style.display = "none";

    if (window.ggChart) {
      window.ggChart.destroy();
      window.ggChart = null;
    }
  }

  // =========================
  // Events
  // =========================
  form.addEventListener("submit", (e) => {
    e.preventDefault();
    if (err) err.style.display = "none";

    const inputs = getInputs();
    if (!inputs.valid) {
      if (err) err.style.display = "block";
      return;
    }

    // ✅ IMPORTANT CHANGE:
    // Track only AFTER validation passes (so it doesn't fire on invalid clicks)
    segmentTrack("User Viewed Calculator Results", SEGMENT_PROPS);

    showOutputs(computeModelB(inputs));
  });

  if (backBtn) backBtn.addEventListener("click", showInputs);

  if (resetBtn) {
    resetBtn.addEventListener("click", () => {
      if (weeklyApptsEl) weeklyApptsEl.value = 33;
      if (staffCountEl) staffCountEl.value = 3;
      if (avgTicketEl) avgTicketEl.value = 100;
      if (pctComeBackEl) pctComeBackEl.value = 70;
      if (err) err.style.display = "none";
      showInputs();
    });
  }

  showInputs();
})();
</script>
</script>
