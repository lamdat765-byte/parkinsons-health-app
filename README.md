const STORAGE_KEYS = {
  accounts: "parkinson_accounts",
  currentUser: "parkinson_current_user",
  patientData: "parkinson_patient_data",
};

const demoUser = {
  name: "Ava Thompson",
  email: "demo@parkinsoncare.app",
  password: "demo123",
};

const defaultPatientData = {
  name: "Ava Thompson",
  symptoms: [
    {
      id: 1,
      date: new Date().toISOString(),
      tremor: 3,
      stiffness: 2,
      balance: 2,
      fatigue: 4,
      notes: "Feeling steady this morning with mild stiffness later in the day.",
    },
    {
      id: 2,
      date: new Date(Date.now() - 86400000).toISOString(),
      tremor: 4,
      stiffness: 3,
      balance: 3,
      fatigue: 5,
      notes: "Balance was more difficult after walking outside.",
    },
    {
      id: 3,
      date: new Date(Date.now() - 172800000).toISOString(),
      tremor: 2,
      stiffness: 2,
      balance: 1,
      fatigue: 3,
      notes: "Morning was good. Less tremor than usual.",
    },
    {
      id: 4,
      date: new Date(Date.now() - 259200000).toISOString(),
      tremor: 4,
      stiffness: 4,
      balance: 2,
      fatigue: 6,
      notes: "Fatigue increased after lunch and arm stiffness came back later in the evening.",
    },
  ],
  medications: [
    { id: 1, name: "Carbidopa/Levodopa", dosage: "25 mg", time: "08:00", note: "Take with breakfast", done: true },
    { id: 2, name: "Carbidopa/Levodopa", dosage: "25 mg", time: "12:30", note: "Before lunch", done: false },
    { id: 3, name: "Vitamin D", dosage: "1000 IU", time: "19:00", note: "With dinner", done: false },
  ],
  exercises: [
    { id: 1, name: "Gentle stretching", duration: "10 min", completed: true },
    { id: 2, name: "Balance walk", duration: "15 min", completed: false },
    { id: 3, name: "Seated arm circles", duration: "5 min", completed: false },
  ],
  notes: "Check in with neurologist about fatigue spikes after lunch. Continue with balance walk and stretching plan.",
};

const state = {
  accounts: [],
  currentUser: null,
  patientData: null,
  authMode: "signin",
  chart: null,
};

const authScreen = document.getElementById("authScreen");
const appShell = document.getElementById("appShell");
const authForm = document.getElementById("authForm");
const authSubmitBtn = document.getElementById("authSubmitBtn");
const authMessage = document.getElementById("authMessage");
const signupFields = document.getElementById("signupFields");
const welcomeName = document.getElementById("welcomeName");
const fullNameInput = document.getElementById("fullName");
const emailInput = document.getElementById("email");
const passwordInput = document.getElementById("password");
const logoutBtn = document.getElementById("logoutBtn");
const saveNotesBtn = document.getElementById("saveNotesBtn");
const careNotes = document.getElementById("careNotes");

function getStoredAccounts() {
  const saved = localStorage.getItem(STORAGE_KEYS.accounts);
  if (!saved) return [];
  try {
    return JSON.parse(saved);
  } catch {
    return [];
  }
}

function saveAccounts(accounts) {
  localStorage.setItem(STORAGE_KEYS.accounts, JSON.stringify(accounts));
}

function initializeDemoAccount() {
  const accounts = getStoredAccounts();
  if (!accounts.length) {
    saveAccounts([demoUser]);
  }
}

function loadCurrentUser() {
  const currentUser = localStorage.getItem(STORAGE_KEYS.currentUser);
  return currentUser ? JSON.parse(currentUser) : null;
}

function saveCurrentUser(user) {
  localStorage.setItem(STORAGE_KEYS.currentUser, JSON.stringify(user));
}

function getPatientData(email) {
  const data = localStorage.getItem(`${STORAGE_KEYS.patientData}_${email}`);
  if (!data) {
    return structuredClone(defaultPatientData);
  }

  try {
    return JSON.parse(data);
  } catch {
    return structuredClone(defaultPatientData);
  }
}

function savePatientData(email, patientData) {
  localStorage.setItem(`${STORAGE_KEYS.patientData}_${email}`, JSON.stringify(patientData));
}

function ensureUserAccount(userEmail) {
  const data = getPatientData(userEmail);
  if (!data || !data.symptoms) {
    savePatientData(userEmail, structuredClone(defaultPatientData));
  }
}

function setAuthMode(mode) {
  state.authMode = mode;
  const tabs = document.querySelectorAll(".toggle-tab");
  tabs.forEach((tab) => tab.classList.toggle("active", tab.dataset.mode === mode));
  signupFields.classList.toggle("hidden", mode !== "signup");
  authSubmitBtn.textContent = mode === "signup" ? "Create account" : "Sign in";
  authMessage.textContent = "";
}

function showAuthScreen() {
  authScreen.classList.remove("hidden");
  appShell.classList.add("hidden");
}

function showAppScreen() {
  authScreen.classList.add("hidden");
  appShell.classList.remove("hidden");
}

function validateCredentials(email, password) {
  const accounts = getStoredAccounts();
  const user = accounts.find(
    (account) => account.email.toLowerCase() === email.toLowerCase() && account.password === password
  );
  return user || null;
}

function handleAuthSubmit(event) {
  event.preventDefault();
  const email = emailInput.value.trim();
  const password = passwordInput.value.trim();
  const name = fullNameInput.value.trim();

  if (!email || !password) {
    authMessage.textContent = "Please fill in all required fields.";
    return;
  }

  if (state.authMode === "signup") {
    const accounts = getStoredAccounts();
    const exists = accounts.some((account) => account.email.toLowerCase() === email.toLowerCase());

    if (exists) {
      authMessage.textContent = "An account with that email already exists.";
      return;
    }

    const newAccount = { name: name || "Patient user", email, password };
    accounts.push(newAccount);
    saveAccounts(accounts);

    state.currentUser = newAccount;
    saveCurrentUser(newAccount);
    savePatientData(email, {
      ...structuredClone(defaultPatientData),
      name: newAccount.name,
      notes: "New patient account created. Add your first symptom log today.",
    });
    renderApp();
    return;
  }

  const user = validateCredentials(email, password);
  if (!user) {
    authMessage.textContent = "Incorrect email or password.";
    return;
  }

  state.currentUser = user;
  saveCurrentUser(user);
  ensureUserAccount(user.email);
  renderApp();
}

function handleLogout() {
  state.currentUser = null;
  localStorage.removeItem(STORAGE_KEYS.currentUser);
  authMessage.textContent = "You have been signed out.";
  authForm.reset();
  showAuthScreen();
}

function formatDate(dateString) {
  return new Intl.DateTimeFormat("en-US", {
    month: "short",
    day: "numeric",
    hour: "numeric",
    minute: "2-digit",
  }).format(new Date(dateString));
}

function calculateAverageSymptom(patientData) {
  if (!patientData.symptoms.length) return 0;
  const total = patientData.symptoms.reduce(
    (sum, item) => sum + item.tremor + item.stiffness + item.balance + item.fatigue,
    0
  );
  return total / (patientData.symptoms.length * 4);
}

function calculateWellness(patientData) {
  const avg = calculateAverageSymptom(patientData);
  const score = Math.max(0, Math.min(100, 100 - avg * 7.5));
  return Math.round(score);
}

function updateOverview(patientData) {
  const wellness = calculateWellness(patientData);
  const completedMeds = patientData.medications.filter((item) => item.done).length;
  const completedExercises = patientData.exercises.filter((item) => item.completed).length;
  const avg = calculateAverageSymptom(patientData);
  const loadText = avg > 6 ? "High" : avg > 3 ? "Moderate" : "Low";

  document.getElementById("wellnessScore").textContent = `${wellness}%`;
  document.getElementById("medCount").textContent = `${completedMeds} / ${patientData.medications.length}`;
  document.getElementById("exerciseCount").textContent = `${completedExercises}`;
  document.getElementById("symptomLoad").textContent = loadText;

  const upcoming = [...patientData.medications]
    .sort((a, b) => a.time.localeCompare(b.time))
    .slice(0, 4)
    .map(
      (item) => `
        <li>
          <span class="schedule-time">${item.time}</span>
          <span>${item.name}</span>
          <span class="status-pill ${item.done ? "done" : "pending"}">${item.done ? "Taken" : "Pending"}</span>
        </li>
      `
    )
    .join("");

  document.getElementById("scheduleList").innerHTML = upcoming || "<li><span>No reminders added.</span></li>";
}

function getChartLabels(patientData) {
  const lastSeven = [...patientData.symptoms]
    .sort((a, b) => new Date(a.date) - new Date(b.date))
    .slice(-7)
    .map((item) => new Date(item.date).toLocaleDateString("en-US", { month: "short", day: "numeric" }));

  return lastSeven.length ? lastSeven : ["No data"];
}

function getChartValues(patientData) {
  const sorted = [...patientData.symptoms].sort((a, b) => new Date(a.date) - new Date(b.date)).slice(-7);
  return sorted.length
    ? sorted.map((item) => {
        const total = item.tremor + item.stiffness + item.balance + item.fatigue;
        return Math.round(total / 4);
      })
    : [0];
}

function renderChart(patientData) {
  const canvas = document.getElementById("symptomsChart");
  const ctx = canvas.getContext("2d");
  const labels = getChartLabels(patientData);
  const values = getChartValues(patientData);

  if (state.chart) {
    state.chart.destroy();
  }

  state.chart = new Chart(ctx, {
    type: "line",
    data: {
      labels,
      datasets: [
        {
          label: "Symptom score",
          data: values,
          borderColor: "#4d6bff",
          backgroundColor: "rgba(77, 107, 255, 0.16)",
          fill: true,
          tension: 0.4,
          pointRadius: 4,
          pointBackgroundColor: "#4d6bff",
        },
      ],
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      plugins: { legend: { display: false } },
      scales: {
        y: {
          min: 0,
          max: 10,
          ticks: { stepSize: 2 },
        },
      },
    },
  });
}

function renderSymptomHistory(patientData) {
  if (!patientData.symptoms.length) {
    document.getElementById("symptomHistory").innerHTML = '<div class="history-entry"><p>No symptom logs yet.</p></div>';
    return;
  }

  const sorted = [...patientData.symptoms].sort((a, b) => new Date(b.date) - new Date(a.date));
  document.getElementById("symptomHistory").innerHTML = sorted
    .map((entry) => {
      const avg = Math.round((entry.tremor + entry.stiffness + entry.balance + entry.fatigue) / 4);
      return `
        <div class="history-entry">
          <div>
            <strong>${formatDate(entry.date)}</strong>
            <div class="meta">${entry.notes || "No notes recorded."}</div>
          </div>
          <div class="meta">Avg: ${avg}/10</div>
          <div class="score-pill">${avg}</div>
        </div>
      `;
    })
    .join("");
}

function renderMedications(patientData) {
  const container = document.getElementById("medicationList");
  if (!patientData.medications.length) {
    container.innerHTML = '<div class="medication-entry"><p>No medications added.</p></div>';
    return;
  }

  container.innerHTML = patientData.medications
    .sort((a, b) => a.time.localeCompare(b.time))
    .map(
      (item) => `
        <div class="medication-entry">
          <div>
            <strong>${item.name}</strong>
            <div class="meta">${item.dosage || "Dose not set"} • ${item.time}</div>
            <div class="meta">${item.note || "No extra note"}</div>
          </div>
          <button data-id="${item.id}" class="toggle-medication">${item.done ? "Mark undone" : "Mark done"}</button>
        </div>
      `
    )
    .join("");

  document.querySelectorAll(".toggle-medication").forEach((button) => {
    button.addEventListener("click", () => {
      const id = Number(button.dataset.id);
      patientData.medications = patientData.medications.map((item) =>
        item.id === id ? { ...item, done: !item.done } : item
      );
      savePatientData(state.currentUser.email, patientData);
      renderApp();
    });
  });
}

function renderExercises(patientData) {
  const container = document.getElementById("exerciseList");
  container.innerHTML = patientData.exercises
    .map(
      (exercise) => `
        <article class="exercise-card">
          <div class="panel-header">
            <h4>${exercise.name}</h4>
            <span class="status-pill ${exercise.completed ? "done" : "pending"}">${exercise.completed ? "Done" : "Pending"}</span>
          </div>
          <p>${exercise.duration}</p>
          <button data-id="${exercise.id}" class="toggle-exercise">${exercise.completed ? "Reset" : "Complete"}</button>
        </article>
      `
    )
    .join("");

  document.querySelectorAll(".toggle-exercise").forEach((button) => {
    button.addEventListener("click", () => {
      const id = Number(button.dataset.id);
      patientData.exercises = patientData.exercises.map((exercise) =>
        exercise.id === id ? { ...exercise, completed: !exercise.completed } : exercise
      );
      savePatientData(state.currentUser.email, patientData);
      renderApp();
    });
  });
}

function renderCareNotes(patientData) {
  careNotes.value = patientData.notes || "";
}

function renderApp() {
  const user = loadCurrentUser();
  if (!user) {
    showAuthScreen();
    return;
  }

  state.currentUser = user;
  const patientData = getPatientData(user.email);
  state.patientData = patientData;

  document.getElementById("welcomeName").textContent = `Welcome, ${user.name}`;
  updateOverview(patientData);
  renderChart(patientData);
  renderSymptomHistory(patientData);
  renderMedications(patientData);
  renderExercises(patientData);
  renderCareNotes(patientData);
  showAppScreen();
}

function createSymptomEntry(formData) {
  return {
    id: Date.now(),
    date: new Date().toISOString(),
    tremor: Number(formData.get("tremor")),
    stiffness: Number(formData.get("stiffness")),
    balance: Number(formData.get("balance")),
    fatigue: Number(formData.get("fatigue")),
    notes: String(formData.get("notes") || ""),
  };
}

document.querySelectorAll(".toggle-tab").forEach((tab) => {
  tab.addEventListener("click", () => setAuthMode(tab.dataset.mode));
});

authForm.addEventListener("submit", handleAuthSubmit);
logoutBtn.addEventListener("click", handleLogout);

const symptomForm = document.getElementById("symptomForm");
symptomForm.addEventListener("submit", (event) => {
  event.preventDefault();
  const user = loadCurrentUser();
  if (!user) return;

  const patientData = getPatientData(user.email);
  patientData.symptoms = [createSymptomEntry(new FormData(symptomForm)), ...patientData.symptoms].slice(0, 30);
  savePatientData(user.email, patientData);
  symptomForm.reset();
  renderApp();
});

const medicationForm = document.getElementById("medicationForm");
medicationForm.addEventListener("submit", (event) => {
  event.preventDefault();
  const user = loadCurrentUser();
  if (!user) return;

  const formData = new FormData(medicationForm);
  const name = String(formData.get("name") || "").trim();
  const time = String(formData.get("time") || "");
  if (!name || !time) return;

  const patientData = getPatientData(user.email);
  patientData.medications.push({
    id: Date.now(),
    name,
    dosage: String(formData.get("dosage") || ""),
    time,
    note: String(formData.get("note") || ""),
    done: false,
  });

  savePatientData(user.email, patientData);
  medicationForm.reset();
  renderApp();
});

const exerciseForm = document.getElementById("exerciseForm");
exerciseForm.addEventListener("submit", (event) => {
  event.preventDefault();
  const user = loadCurrentUser();
  if (!user) return;

  const formData = new FormData(exerciseForm);
  const name = String(formData.get("exerciseName") || "").trim();
  const duration = String(formData.get("duration") || "").trim();
  if (!name || !duration) return;

  const patientData = getPatientData(user.email);
  patientData.exercises.push({ id: Date.now(), name, duration, completed: false });
  savePatientData(user.email, patientData);
  exerciseForm.reset();
  renderApp();
});

saveNotesBtn.addEventListener("click", () => {
  const user = loadCurrentUser();
  if (!user) return;

  const patientData = getPatientData(user.email);
  patientData.notes = careNotes.value;
  savePatientData(user.email, patientData);
  saveNotesBtn.textContent = "Saved";
  setTimeout(() => {
    saveNotesBtn.textContent = "Save notes";
  }, 1200);
});

document.querySelectorAll(".nav-item").forEach((button) => {
  button.addEventListener("click", () => {
    document.querySelectorAll(".nav-item").forEach((item) => item.classList.remove("active"));
    button.classList.add("active");
    document.querySelectorAll(".section").forEach((section) => section.classList.remove("active"));
    document.getElementById(button.dataset.section).classList.add("active");
  });
});

initializeDemoAccount();
const user = loadCurrentUser();
if (user) {
  ensureUserAccount(user.email);
  renderApp();
} else {
  setAuthMode("signin");
  showAuthScreen();
}

window.addEventListener("resize", () => {
  if (state.currentUser) {
    renderApp();
  }
});

window.addEventListener("beforeunload", () => {
  if (state.currentUser) {
    saveCurrentUser(state.currentUser);
  }
});











































			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			

			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			

			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			t
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			
			

			
			
			
			
			
			
			
			
			
			
