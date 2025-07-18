<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Attendance App</title>
  <script src="https://cdn.jsdelivr.net/npm/qrcodejs@1.0.0/qrcode.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/html2canvas@1.4.1/dist/html2canvas.min.js"></script>
  <script src="https://unpkg.com/html5-qrcode"></script>
  <style>
    body {
      font-family: Arial, sans-serif;
      margin: 0;
      padding: 0;
      background: #f5f5f5;
    }
    .container {
      max-width: 500px;
      margin: auto;
      background: white;
      padding: 20px;
      box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
      margin-top: 30px;
    }
    h1, h2 {
      text-align: center;
    }
    select, input, button {
      width: 100%;
      padding: 10px;
      margin-top: 10px;
      box-sizing: border-box;
    }
    #idCard {
      display: none;
      margin-top: 20px;
      text-align: center;
    }
    #attendanceSection {
      display: none;
    }
    #adminSidebar {
      height: 100%;
      width: 0;
      position: fixed;
      top: 0;
      right: 0;
      background-color: white;
      overflow-x: hidden;
      overflow-y: auto;
      transition: width 0.5s ease, padding 0.3s ease;
      box-shadow: -2px 0 5px rgba(0,0,0,0.5);
      z-index: 1000;
      padding: 0;
    }
    #adminSidebar.open {
      width: 300px;
      padding: 20px;
    }
    .admin-toggle {
      background: #333;
      color: white;
      padding: 10px;
      cursor: pointer;
      text-align: center;
      margin-top: 20px;
    }
    table {
      width: 100%;
      border-collapse: collapse;
      margin-top: 10px;
    }
    th, td {
      border: 1px solid #ddd;
      padding: 8px;
      text-align: left;
    }
    th {
      background: #eee;
    }
    .admin-section {
      margin-top: 20px;
    }
  </style>
</head>
<body>
<div class="container">
  <h1>Attendance App</h1>
  <h2>Sign In / Register</h2>
  <select id="nameSelect"></select>
  <input type="text" id="nameInput" placeholder="Or enter your name">
  <input type="text" id="mobileInput" placeholder="Mobile Number">
  <input type="text" id="emailInput" placeholder="Email">
  <input type="text" id="addressInput" placeholder="Address">
  <button onclick="signInOrRegister()">Continue</button>

  <div id="idCard"></div>

  <div id="attendanceSection">
    <h2>Mark Attendance</h2>
    <button onclick="markAttendance()">Mark Attendance</button>
    <div id="attendanceMsg" style="margin-top:10px; color: green;"></div>
  </div>

  <div class="admin-toggle" onclick="promptPassword()">Admin Panel</div>
</div>

<div id="adminSidebar">
  <button onclick="closeAdminPanel()" style="width: 100%; background: #d9534f; color: white; border: none; padding: 10px; margin-bottom: 15px; cursor: pointer;">
    Close Admin Panel
  </button>
  <h2>Admin Panel</h2>
  <button onclick="loadData()">Load Attendance</button>
  <button onclick="exportCSV()">Export CSV</button>
  <button onclick="listUsers()">List Users</button>
  <button onclick="resetUsers()">Clear All Users</button>
  <button onclick="resetAttendance()">Clear Attendance</button>
  <button onclick="exportUsers()">Export Users</button>
  <table id="dataTable"></table>
  <table id="usersTable"></table>

  <div class="admin-section">
    <h3>QR Code Attendance Scanner</h3>
    <div id="qr-reader" style="width: 100%;"></div>
    <div id="qr-reader-results" style="margin-top: 10px; color: green;"></div>
    <button onclick="startQrScanner()">Start Scanning</button>
    <button onclick="stopQrScanner()">Stop Scanning</button>
  </div>

  <div class="admin-section">
    <h3>Change Admin Password</h3>
    <input type="password" id="newPasswordInput" placeholder="Enter new password" />
    <button onclick="changeAdminPassword()">Update Password</button>
  </div>
</div>

<script>
const usersKey = "all_users";
const currentUserKey = "current_user";
const storageKey = "attendance_data";
let adminPassword = localStorage.getItem("adminPassword") || "admin123";
let currentUser = null;
let qrScanner;

function promptPassword() {
  const pass = prompt("Enter admin password:");
  const panel = document.getElementById("adminSidebar");
  if (pass === adminPassword) {
    panel.classList.add("open");
  } else {
    alert("Incorrect password");
  }
}

function closeAdminPanel() {
  stopQrScanner();
  document.getElementById("adminSidebar").classList.remove("open");
}

function changeAdminPassword() {
  const newPass = document.getElementById("newPasswordInput").value;
  if (!newPass) return alert("Enter a new password");
  localStorage.setItem("adminPassword", newPass);
  adminPassword = newPass;
  document.getElementById("newPasswordInput").value = "";
  alert("Admin password updated successfully");
}

function populateNameDropdown() {
  const select = document.getElementById("nameSelect");
  const users = JSON.parse(localStorage.getItem(usersKey) || "[]");
  select.innerHTML = '<option value="">-- Select your name --</option>';
  users.forEach(u => {
    const opt = document.createElement("option");
    opt.value = u.name;
    opt.text = u.displayName;
    select.appendChild(opt);
  });
}

function generateUserId(index) {
  return `Pro-Mum-${String(index + 1).padStart(5, '0')}`;
}

function signInOrRegister() {
  const nameInput = document.getElementById("nameInput").value.trim();
  const nameSelect = document.getElementById("nameSelect").value;
  const name = nameInput || nameSelect;
  if (!name) return alert("Enter or select your name");

  const mobile = document.getElementById("mobileInput").value.trim();
  const email = document.getElementById("emailInput").value.trim();
  const address = document.getElementById("addressInput").value.trim();

  let users = JSON.parse(localStorage.getItem(usersKey) || "[]");
  let user = users.find(u => u.name.toLowerCase() === name.toLowerCase());

  if (!user) {
    user = {
      name: name.toLowerCase(),
      displayName: name,
      userId: generateUserId(users.length),
      mobile, email, address
    };
    users.push(user);
    localStorage.setItem(usersKey, JSON.stringify(users));
  }

  currentUser = user;
  localStorage.setItem(currentUserKey, JSON.stringify(currentUser));
  populateNameDropdown();
  showIdCard();
  document.getElementById("attendanceSection").style.display = "block";
  document.getElementById("attendanceMsg").innerText = "";
}

function showIdCard() {
  const card = document.getElementById("idCard");
  const u = currentUser;

  card.innerHTML = `
    <div id="idCardWrapper" style="
      display: flex;
      justify-content: space-between;
      align-items: center;
      border: 2px solid #000;
      padding: 20px;
      background: white;
      width: 100%;
      max-width: 400px;
      margin: auto;
    ">
      <div style="flex: 1; padding-right: 10px; text-align: left;">
        <h3 style="margin-top: 0;">ID Card</h3>
        <p><strong>Name:</strong> ${u.displayName}</p>
        <p><strong>User ID:</strong> ${u.userId}</p>
        <p><strong>Mobile:</strong> ${u.mobile}</p>
        <p><strong>Email:</strong> ${u.email}</p>
        <p><strong>Address:</strong> ${u.address}</p>
      </div>
      <div id="qrcode"></div>
    </div>
    <button onclick="downloadIdCard()" style="margin-top: 10px;">Download ID Card</button>
  `;

  card.style.display = "block";
  document.getElementById("qrcode").innerHTML = "";
  new QRCode(document.getElementById("qrcode"), {
    text: u.userId,
    width: 128,
    height: 128
  });
}

function downloadIdCard() {
  const cardElement = document.getElementById("idCardWrapper");
  html2canvas(cardElement).then(canvas => {
    const link = document.createElement("a");
    link.download = `${currentUser.userId}_id_card.png`;
    link.href = canvas.toDataURL("image/png");
    link.click();
  });
}

function markAttendance() {
  if (!currentUser) return alert("Please sign in first.");
  const record = {
    userId: currentUser.userId,
    name: currentUser.displayName,
    timestamp: new Date().toISOString()
  };
  const data = JSON.parse(localStorage.getItem(storageKey) || "[]");
  data.push(record);
  localStorage.setItem(storageKey, JSON.stringify(data));
  document.getElementById("attendanceMsg").innerText = "Attendance marked successfully!";
}

function markAttendanceFromQr(userId) {
  const users = JSON.parse(localStorage.getItem(usersKey) || "[]");
  const user = users.find(u => u.userId === userId);
  if (!user) {
    alert("User not found!");
    return;
  }

  const record = {
    userId: user.userId,
    name: user.displayName,
    timestamp: new Date().toISOString()
  };

  const data = JSON.parse(localStorage.getItem(storageKey) || "[]");
  data.push(record);
  localStorage.setItem(storageKey, JSON.stringify(data));
  alert(`Attendance marked for ${user.displayName}`);
  loadData();
}

function startQrScanner() {
  const qrResult = document.getElementById("qr-reader-results");
  qrResult.innerText = "Initializing camera...";

  navigator.mediaDevices.getUserMedia({ video: true })
    .then(stream => {
      stream.getTracks().forEach(track => track.stop());
      qrScanner = new Html5Qrcode("qr-reader");
      qrScanner.start(
        { facingMode: "environment" },
        {
          fps: 10,
          qrbox: 200
        },
        qrCodeMessage => {
          qrResult.innerText = `Scanned User ID: ${qrCodeMessage}`;
          markAttendanceFromQr(qrCodeMessage);
          stopQrScanner();
        },
        errorMessage => {
          // Optional: console.log(errorMessage);
        }
      ).catch(err => {
        qrResult.innerText = `Error: ${err}`;
      });
    })
    .catch(err => {
      qrResult.innerText = `Camera access denied: ${err.message}`;
      alert("Camera access is required. Please allow camera permission.");
    });
}

function stopQrScanner() {
  if (qrScanner) {
    qrScanner.stop().then(() => {
      document.getElementById("qr-reader").innerHTML = "";
      document.getElementById("qr-reader-results").innerText = "Scanner stopped.";
    }).catch(err => {
      console.error("Error stopping scanner", err);
    });
  }
}

function loadData() {
  const data = JSON.parse(localStorage.getItem(storageKey) || "[]");
  const table = document.getElementById("dataTable");
  if (data.length === 0) {
    table.innerHTML = "<tr><td>No records found.</td></tr>";
    return;
  }
  const header = "<tr><th>Name</th><th>User ID</th><th>Timestamp</th></tr>";
  const rows = data.map(d => `<tr><td>${d.name}</td><td>${d.userId}</td><td>${new Date(d.timestamp).toLocaleString()}</td></tr>`).join("");
  table.innerHTML = header + rows;
}

function exportCSV() {
  const data = JSON.parse(localStorage.getItem(storageKey) || "[]");
  const headers = ["Name", "User ID", "Timestamp"];
  const rows = data.map(d => [d.name, d.userId, d.timestamp]);
  const csv = [headers, ...rows].map(r => r.join(",")).join("\n");
  const blob = new Blob([csv], { type: "text/csv;charset=utf-8;" });
  const url = URL.createObjectURL(blob);
  const link = document.createElement("a");
  link.setAttribute("href", url);
  link.setAttribute("download", "attendance_data.csv");
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
}

function listUsers() {
  const users = JSON.parse(localStorage.getItem(usersKey) || "[]");
  const table = document.getElementById("usersTable");
  if (users.length === 0) {
    table.innerHTML = "<tr><td>No users found.</td></tr>";
    return;
  }
  const header = "<tr><th>Name</th><th>User ID</th><th>Mobile</th><th>Email</th><th>Address</th></tr>";
  const rows = users.map(u => `<tr><td>${u.displayName}</td><td>${u.userId}</td><td>${u.mobile}</td><td>${u.email}</td><td>${u.address}</td></tr>`).join("");
  table.innerHTML = header + rows;
}

function resetUsers() {
  if (confirm("Are you sure you want to delete all users?")) {
    localStorage.removeItem(usersKey);
    document.getElementById("usersTable").innerHTML = "";
    populateNameDropdown();
    alert("All users deleted.");
  }
}

function resetAttendance() {
  if (confirm("Are you sure you want to delete all attendance records?")) {
    localStorage.removeItem(storageKey);
    document.getElementById("dataTable").innerHTML = "";
    alert("All attendance records deleted.");
  }
}

function exportUsers() {
  const users = JSON.parse(localStorage.getItem(usersKey) || "[]");
  const headers = ["Name", "User ID", "Mobile", "Email", "Address"];
  const rows = users.map(u => [u.displayName, u.userId, u.mobile, u.email, u.address]);
  const csv = [headers, ...rows].map(r => r.join(",")).join("\n");
  const blob = new Blob([csv], { type: "text/csv;charset=utf-8;" });
  const url = URL.createObjectURL(blob);
  const link = document.createElement("a");
  link.setAttribute("href", url);
  link.setAttribute("download", "user_list.csv");
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
}

window.onload = populateNameDropdown;
</script>
</body>
</html>
