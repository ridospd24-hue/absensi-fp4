<!DOCTYPE html>
<html lang="id">
<head>
  <meta charset="UTF-8">
  <title>Aplikasi Absensi - Kelas FASE FP 4</title>
  <script src="https://unpkg.com/html5-qrcode" type="text/javascript"></script>
  <style>
    body { font-family: Poppins, sans-serif; margin:0; padding:0; background:#f5faff; color:#333; }
    header { background:#3A9BDC; color:white; padding:1rem; text-align:center; }
    .container { padding:1.5rem; }
    .card { background:white; padding:1.5rem; border-radius:12px; box-shadow:0 4px 10px rgba(0,0,0,0.1); margin-bottom:1.5rem; }
    input, button, select { padding:0.5rem; border-radius:8px; border:1px solid #ccc; margin-top:0.5rem; width:100%; }
    button { background:#3A9BDC; color:white; border:none; cursor:pointer; }
    button:hover { background:#2e82b8; }
    .qr-container { text-align:center; }
    .status { margin-top:1rem; font-weight:bold; }
  </style>
</head>
<body>
<header>
  <h1>Kelas FASE FP 4</h1>
  <p>Absensi Digital QR Code</p>
</header>
<div class="container" id="app">
  <!-- Halaman Pilih Login -->
  <div class="card" id="login-choice">
    <h2>Pilih Mode Login</h2>
    <button onclick="showLogin('siswa')">Login Siswa</button>
    <button onclick="showLogin('sekre')">Login Sekretaris</button>
    <button onclick="showLogin('guru')">Login Guru</button>
  </div>

  <!-- Login Form -->
  <div class="card" id="login-form" style="display:none;">
    <h2 id="login-title"></h2>
    <input type="text" id="username" placeholder="Username">
    <input type="password" id="password" placeholder="Password">
    <button onclick="login()">Login</button>
    <button onclick="backToChoice()">Kembali</button>
  </div>

  <!-- Halaman QR Code Siswa -->
  <div class="card" id="siswa-page" style="display:none;">
    <h2>QR Code Anda</h2>
    <div class="qr-container">
      <div id="siswa-qr"></div>
      <p id="siswa-name"></p>
    </div>
    <button onclick="logout()">Logout</button>
  </div>

  <!-- Halaman Sekretaris Scan QR -->
  <div class="card" id="sekre-page" style="display:none;">
    <h2>Scan QR Siswa</h2>
    <div id="reader" style="width:300px; margin:auto;"></div>
    <div id="scan-result" class="status"></div>
    <div id="absen-form" style="display:none;">
      <p id="scanned-name"></p>
      <select id="status-select">
        <option value="Hadir">✔ Hadir</option>
        <option value="Sakit">🟡 Sakit</option>
        <option value="Alfa">✖ Alfa</option>
      </select>
      <button onclick="simpanAbsensi()">Simpan Absensi</button>
    </div>
    <button onclick="logout()">Logout</button>
  </div>

  <!-- Halaman Guru -->
  <div class="card" id="guru-page" style="display:none;">
    <h2>Dashboard Rekap Absensi</h2>
    <div id="rekap"></div>
    <button onclick="downloadExcel()">Download Excel</button>
    <button onclick="logout()">Logout</button>
  </div>
</div>

<script>
  let currentRole = null;
  let siswaList = {
    "ANANDA DESRIANA": "1",
    "BUTET MAHARANI MANALU": "2",
    "DANIEL SABATH SARAGIH": "3"
    // Tambahkan semua siswa dari PASSWORD.xlsx sesuai format "NAMA": "PASSWORD"
  };
  let absensi = [];
  let html5QrCode;

  function showLogin(role){
    currentRole = role;
    document.getElementById('login-choice').style.display='none';
    document.getElementById('login-form').style.display='block';
    document.getElementById('login-title').innerText = 'Login ' + role.toUpperCase();
  }

  function backToChoice(){
    document.getElementById('login-form').style.display='none';
    document.getElementById('login-choice').style.display='block';
  }

  function login(){
    let u = document.getElementById('username').value.trim();
    let p = document.getElementById('password').value.trim();
    if(currentRole==='siswa'){
      if(siswaList[u] && siswaList[u]===p){
        document.getElementById('login-form').style.display='none';
        document.getElementById('siswa-page').style.display='block';
        document.getElementById('siswa-name').innerText=u;
        let qr = `https://chart.googleapis.com/chart?cht=qr&chs=200x200&chl=${encodeURIComponent(u)}`;
        document.getElementById('siswa-qr').innerHTML = `<img src="${qr}"/>`;
      } else alert('Login gagal');
    }
    if(currentRole==='sekre'){
      if(u==='SEKRE' && p==='SEKREFP4'){
        document.getElementById('login-form').style.display='none';
        document.getElementById('sekre-page').style.display='block';
        startScanner();
      } else alert('Login gagal');
    }
    if(currentRole==='guru'){
      if(u==='RIDO' && p==='RIDO'){
        document.getElementById('login-form').style.display='none';
        document.getElementById('guru-page').style.display='block';
        renderRekap();
      } else alert('Login gagal');
    }
  }

  function logout(){
    if(html5QrCode){
      html5QrCode.stop().catch(err=>console.log(err));
    }
    document.querySelectorAll('.card').forEach(c=>c.style.display='none');
    document.getElementById('login-choice').style.display='block';
  }

  function startScanner(){
    html5QrCode = new Html5Qrcode("reader");
    const config = { fps: 10, qrbox: 250 };
    html5QrCode.start({ facingMode: "environment" }, config,
      qrCodeMessage => {
        document.getElementById('scan-result').innerText = "QR Terdeteksi: "+qrCodeMessage;
        if(siswaList[qrCodeMessage]){
          document.getElementById('absen-form').style.display='block';
          document.getElementById('scanned-name').innerText = qrCodeMessage;
        }
      },
      errorMessage => {
      }
    ).catch(err=>console.log(err));
  }

  function simpanAbsensi(){
    let nama = document.getElementById('scanned-name').innerText;
    let status = document.getElementById('status-select').value;
    absensi.push({nama, status, waktu: new Date().toLocaleString()});
    alert('Absensi tersimpan: '+nama+" - "+status);
    document.getElementById('absen-form').style.display='none';
    renderRekap();
  }

  function renderRekap(){
    let div = document.getElementById('rekap');
    if(!div) return;
    div.innerHTML = '<h3>Rekap Absensi</h3>';
    let table = '<table border="1" cellpadding="5"><tr><th>Nama</th><th>Status</th><th>Waktu</th></tr>';
    absensi.forEach(a=>{
      table += `<tr><td>${a.nama}</td><td>${a.status}</td><td>${a.waktu}</td></tr>`;
    });
    table+='</table>';
    div.innerHTML += table;
  }

  function downloadExcel(){
    let csv = "Nama,Status,Waktu\n";
    absensi.forEach(a=>{
      csv += `${a.nama},${a.status},${a.waktu}\n`;
    });
    let blob = new Blob([csv], { type: 'text/csv' });
    let url = window.URL.createObjectURL(blob);
    let a = document.createElement('a');
    a.href = url;
    a.download = 'rekap_absensi.csv';
    a.click();
  }
</script>
</body>
</html>
