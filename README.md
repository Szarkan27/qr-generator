<!DOCTYPE html>
<html lang="pl">
<head>
  <meta charset="UTF-8">
  <title>Generator kodów QR z bazą</title>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/qrcodejs/1.0.0/qrcode.min.js"></script>
  <style>
    body {
      font-family: Arial, sans-serif;
      padding: 20px;
      background-color: #ffffff;
    }

    .tab { display: none; }
    .tab.active { display: block; }

    .tabs {
      text-align: center;
      margin-bottom: 20px;
    }

    .tabs button {
      margin: 0 10px;
      padding: 10px 20px;
      font-weight: bold;
      border: none;
      background-color: #333;
      color: white;
      border-radius: 5px;
      cursor: pointer;
    }

    .tabs button:hover {
      background-color: #555;
    }

    #generator, #baza {
      max-width: 900px;
      margin: 0 auto;
      background-color: #f9f9f9;
      padding: 20px;
      border-radius: 10px;
      box-shadow: 0 4px 20px rgba(0,0,0,0.1);
    }

    label {
      display: block;
      margin-bottom: 10px;
    }

    input[type="text"], textarea {
      width: 100%;
      padding: 8px;
      font-size: 14px;
      border-radius: 4px;
      border: 1px solid #ccc;
      box-sizing: border-box;
    }

    textarea#zoList {
      resize: vertical;
      min-height: 120px;
    }

    button {
      padding: 10px 15px;
      border: none;
      border-radius: 5px;
      background-color: #333;
      color: white;
      cursor: pointer;
    }

    button:hover {
      background-color: #555;
    }

    table {
      border-collapse: collapse;
      margin-top: 10px;
      width: 100%;
      table-layout: auto;
      word-break: break-word;
    }

    th, td {
      border: 1px solid #ccc;
      padding: 5px;
      text-align: center;
      max-width: 300px;
    }

    .small {
      font-size: 10px;
      padding: 2px 5px;
    }

    .qr-container {
      width: 4cm;
      height: 5cm;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      margin-top: 20px;
    }

    #qrcodePrint {
      width: 2.7cm;
      height: 2.7cm;
    }

    #labelPrint {
      margin-top: 0.3cm;
      font-size: 10pt;
      writing-mode: vertical-rl;
      text-align: center;
      max-height: 2.7cm;
      overflow: hidden;
      word-break: break-word;
    }

    @media print {
      @page {
        size: 4cm 5cm;
        margin: 0;
      }

      body {
        visibility: hidden;
        margin: 0;
        padding: 0;
      }

      #printArea {
        visibility: visible;
        position: fixed;
        top: 0;
        left: 0;
        width: 4cm;
        height: 5cm;
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
      }

      #qrcodePrint {
        width: 2.7cm;
        height: 2.7cm;
      }

      #labelPrint {
        font-size: 10pt;
      }
    }
  </style>
</head>
<body>
  <div class="tabs">
    <button onclick="switchTab('generator')">Generator</button>
    <button onclick="switchTab('baza')">Baza danych</button>
  </div>

  <div id="generator" class="tab active">
    <h2>Generator kodów QR</h2>
    <label>RLP:
      <input type="text" id="rlp" />
    </label>
    <label>Lista Zo (wklej z Excela):
      <textarea id="zoList" placeholder="Wklej listę kodów Zo..."></textarea>
    </label>
    <button onclick="generateAndSave()">Generuj i dodaj do bazy</button>
  </div>

  <div id="baza" class="tab">
    <h2>Baza danych</h2>
    <button onclick="copyData()">Kopiuj</button>
    <table>
      <thead>
        <tr>
          <th>RLP</th>
          <th>Zo</th>
          <th>Data</th>
          <th>Akcje</th>
        </tr>
      </thead>
      <tbody id="bazaTable"></tbody>
    </table>
  </div>

  <!-- Ukryty kontener do druku -->
  <div id="printArea" class="qr-container">
    <div id="qrcodePrint"></div>
    <div id="labelPrint"></div>
  </div>

  <script>
    function switchTab(tabId) {
      document.querySelectorAll(".tab").forEach(tab => tab.classList.remove("active"));
      document.getElementById(tabId).classList.add("active");
    }

    function generateAndSave() {
      const rlp = document.getElementById("rlp").value.trim();
      const zoList = document.getElementById("zoList").value.trim().split(/\n+/).filter(Boolean);

      if (!rlp || zoList.length === 0) {
        alert("Wprowadź RLP i listę Zo.");
        return;
      }

      const baza = JSON.parse(localStorage.getItem("qrBaza") || "[]");
      const now = new Date();
      const timestamp = now.getTime();
      const displayDate = now.toLocaleString();

      zoList.forEach(zo => {
        baza.unshift({ rlp, zo, timestamp, displayDate });
      });

      localStorage.setItem("qrBaza", JSON.stringify(baza));
      document.getElementById("rlp").value = "";
      document.getElementById("zoList").value = "";
      renderBaza();
      printQR(rlp);
    }

    function renderBaza() {
      const baza = JSON.parse(localStorage.getItem("qrBaza") || "[]");
      const tbody = document.getElementById("bazaTable");
      tbody.innerHTML = "";

      const now = Date.now();
      const pięćDni = 5 * 24 * 60 * 60 * 1000;

      const aktualna = baza.filter(entry => now - entry.timestamp <= pięćDni);
      localStorage.setItem("qrBaza", JSON.stringify(aktualna));

      aktualna.forEach((item, index) => {
        const tr = document.createElement("tr");
        tr.innerHTML = `
          <td>${item.rlp}</td>
          <td>${item.zo}</td>
          <td>${item.displayDate}</td>
          <td><button class="small" onclick="deleteQR(${index})">Usuń</button></td>
        `;
        tbody.appendChild(tr);
      });
    }

    function deleteQR(index) {
      const baza = JSON.parse(localStorage.getItem("qrBaza") || "[]");
      baza.splice(index, 1);
      localStorage.setItem("qrBaza", JSON.stringify(baza));
      renderBaza();
    }

    function printQR(text) {
      const qrcode = document.getElementById("qrcodePrint");
      const label = document.getElementById("labelPrint");
      qrcode.innerHTML = "";
      label.textContent = text;

      new QRCode(qrcode, {
        text: text,
        width: 102,
        height: 102,
        correctLevel: QRCode.CorrectLevel.H
      });

      setTimeout(() => {
        const original = document.body.innerHTML;
        document.body.innerHTML = document.getElementById("printArea").outerHTML;
        window.print();
        document.body.innerHTML = original;
        switchTab("baza");
      }, 300);
    }

    function copyData() {
      const startInput = prompt("Podaj datę początkową (dd-mm-rrrr):");
      if (!startInput) return;

      const startDate = parseDate(startInput.trim());
      if (!startDate) {
        alert("Nieprawidłowy format daty początkowej.");
        return;
      }

      const endInput = prompt("Podaj datę końcową (dd-mm-rrrr) (opcjonalnie):");
      const endDate = endInput ? parseDate(endInput.trim()) : null;

      const baza = JSON.parse(localStorage.getItem("qrBaza") || "[]");

      const selected = baza.filter(entry => {
        const entryDate = new Date(entry.timestamp);
        const start = new Date(startDate);
        const end = endDate ? new Date(endDate) : new Date(startDate);

        entryDate.setHours(0, 0, 0, 0);
        start.setHours(0, 0, 0, 0);
        end.setHours(23, 59, 59, 999);

        return entryDate >= start && entryDate <= end;
      });

      if (selected.length === 0) {
        alert("Brak danych do skopiowania.");
        return;
      }

      const text = selected.map(item => `${item.rlp}\t${item.zo}\t${item.displayDate}`).join("\n");
      navigator.clipboard.writeText(text).then(() => {
        alert(`Skopiowano ${selected.length} rekordów.`);
      });
    }

    function parseDate(input) {
      const parts = input.split("-");
      if (parts.length !== 3) return null;
      const [day, month, year] = parts.map(Number);
      return new Date(year, month - 1, day);
    }

    renderBaza();
  </script>
</body>
</html>
