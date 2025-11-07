[index.html](https://github.com/user-attachments/files/23423238/index.html)
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="utf-8" />
  <title>Registro de Embolses (Supabase)</title>
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js"></script>
  <style>
    body { font-family: sans-serif; background: #f0fdf4; margin: 0; }
    .container { max-width: 900px; margin: 30px auto; background: #fff; border-radius: 10px; box-shadow: 0 4px 12px #0001; padding: 30px; }
    h1 { color: #16a34a; }
    label { font-weight: bold; }
    input, select { padding: 8px; border-radius: 5px; border: 1px solid #ccc; margin-bottom: 10px; }
    button { padding: 10px 20px; border-radius: 6px; border: none; font-weight: bold; cursor: pointer; }
    .btn-main { background: #16a34a; color: #fff; }
    .btn-sec { background: #2563eb; color: #fff; }
    .btn-danger { background: #dc2626; color: #fff; }
    table { width: 100%; border-collapse: collapse; margin-top: 20px; }
    th, td { border: 1px solid #e5e7eb; padding: 8px; text-align: left; }
    th { background: #f3f4f6; }
    .color-box { display: inline-block; width: 18px; height: 18px; border-radius: 4px; border: 1px solid #666; margin-right: 6px; vertical-align: middle; }
    .flex { display: flex; gap: 10px; align-items: center; }
    .mb { margin-bottom: 20px; }
    .mt { margin-top: 20px; }
    .error { color: #dc2626; font-weight: bold; }
    .success { color: #16a34a; font-weight: bold; }
  </style>
</head>
<body>
  <div class="container" id="app">
    <h1>🍌 Registro de Embolses</h1>
    <div id="main"></div>
  </div>
  <script>
    // === PON AQUÍ TUS CLAVES DE SUPABASE ===
    const SUPABASE_URL = 'https://TU-PROYECTO.supabase.co'; // <-- Cambia esto
    const SUPABASE_KEY = 'TU_ANON_KEY'; // <-- Cambia esto

    const supabase = supabase.createClient(SUPABASE_URL, SUPABASE_KEY);

    // Estado global simple
    let usuarioActual = null;
    let colores = [];
    let embolses = [];
    let usuarios = [];
    let cargando = false;
    let error = '';

    // Utilidad para mostrar mensajes
    function showMsg(msg, type='success') {
      document.getElementById('msg').innerHTML = `<span class="${type}">${msg}</span>`;
      setTimeout(() => { document.getElementById('msg').innerHTML = ''; }, 3000);
    }

    // Renderiza la app
    async function render() {
      const main = document.getElementById('main');
      main.innerHTML = '';
      if (!usuarioActual) {
        main.innerHTML = `
          <div class="mb">
            <label>Usuario:</label><br>
            <input id="login-user" type="text" autocomplete="username"><br>
            <label>Contraseña:</label><br>
            <input id="login-pass" type="password" autocomplete="current-password"><br>
            <button class="btn-main" onclick="login()">Iniciar sesión</button>
            <div id="msg" class="mt"></div>
            <div class="mt" style="font-size:13px;color:#666;">
              <b>Nota:</b> El primer usuario debe ser creado manualmente en Supabase.<br>
              Ejemplo: usuario <b>admin</b> / contraseña <b>admin123</b>
            </div>
          </div>
        `;
        return;
      }

      // Carga datos
      await cargarColores();
      await cargarEmbolses();
      if (usuarioActual.rol === 'admin') await cargarUsuarios();

      // Formulario de embolses
      main.innerHTML = `
        <div class="mb flex" style="justify-content:space-between;">
          <div>
            <b>Usuario:</b> ${usuarioActual.nombre} (${usuarioActual.rol})
          </div>
          <button class="btn-danger" onclick="logout()">Cerrar sesión</button>
        </div>
        <div id="msg"></div>
        <div class="mb">
          <h2>Registrar Embolse</h2>
          <label>Parcela:</label><br>
          <input id="parcela" type="text"><br>
          <label>Cinta:</label><br>
          <select id="cinta">${colores.map(c=>`<option value="${c.nombre}">${c.nombre}</option>`)}</select>
          <span id="color-preview" class="color-box" style="background:${colores[0]?.hex||'#fff'}"></span><br>
          <label>Cantidad:</label><br>
          <input id="cantidad" type="number" min="1"><br>
          <label>Fecha:</label><br>
          <input id="fecha" type="date" value="${new Date().toISOString().slice(0,10)}"><br>
          <button class="btn-main mt" onclick="registrarEmbolse()">Guardar</button>
        </div>
        <div class="mb">
          <h2>Registros</h2>
          <table>
            <thead>
              <tr>
                <th>Fecha</th>
                <th>Parcela</th>
                <th>Cinta</th>
                <th>Cantidad</th>
                <th>Registrado por</th>
                ${usuarioActual.rol==='admin'?'<th>Acciones</th>':''}
              </tr>
            </thead>
            <tbody>
              ${embolses.map(e=>`
                <tr>
                  <td>${e.fecha}</td>
                  <td>${e.parcela}</td>
                  <td><span class="color-box" style="background:${e.color_hex}"></span>${e.cinta}</td>
                  <td>${e.cantidad}</td>
                  <td>${e.usuario_nombre||''}</td>
                  ${usuarioActual.rol==='admin'?`
                  <td>
                    <button class="btn-danger" onclick="eliminarEmbolse('${e.id}')">Eliminar</button>
                  </td>
                  `:''}
                </tr>
              `).join('')}
            </tbody>
          </table>
        </div>
        <div class="mb">
          <button class="btn-sec" onclick="descargarCSV()">Descargar CSV</button>
        </div>
        ${usuarioActual.rol==='admin'?`
        <div class="mb">
          <h2>Colores</h2>
          <form onsubmit="agregarColor(event)">
            <label>Nombre:</label>
            <input id="color-nombre" type="text" required>
            <label>Color:</label>
            <input id="color-hex" type="color" value="#000000" required>
            <button class="btn-main" type="submit">Agregar</button>
          </form>
          <ul>
            ${colores.map(c=>`
              <li>
                <span class="color-box" style="background:${c.hex}"></span>
                ${c.nombre}
                <button class="btn-danger" onclick="eliminarColor('${c.id}')">Eliminar</button>
              </li>
            `).join('')}
          </ul>
        </div>
        <div class="mb">
          <h2>Usuarios</h2>
          <form onsubmit="agregarUsuario(event)">
            <label>Nombre:</label>
            <input id="user-nombre" type="text" required>
            <label>Usuario:</label>
            <input id="user-username" type="text" required>
            <label>Contraseña:</label>
            <input id="user-pass" type="password" required>
            <label>Rol:</label>
            <select id="user-rol">
              <option value="usuario">Usuario</option>
              <option value="admin">Admin</option>
            </select>
            <button class="btn-main" type="submit">Agregar</button>
          </form>
          <ul>
            ${usuarios.map(u=>`
              <li>
                ${u.nombre} (${u.username}) - ${u.rol}
                ${u.id!==usuarioActual.id?`<button class="btn-danger" onclick="eliminarUsuario('${u.id}')">Eliminar</button>`:''}
              </li>
            `).join('')}
          </ul>
        </div>
        `:''}
      `;

      // Color preview
      document.getElementById('cinta').addEventListener('change', function() {
        const color = colores.find(c=>c.nombre===this.value);
        document.getElementById('color-preview').style.background = color?color.hex:'#fff';
      });
    }

    // Login
    async function login() {
      const username = document.getElementById('login-user').value.trim();
      const password = document.getElementById('login-pass').value.trim();
      if (!username || !password) return showMsg('Completa usuario y contraseña','error');
      const { data, error } = await supabase.from('usuarios').select('*').eq('username', username).eq('password', password).single();
      if (error || !data) return showMsg('Usuario o contraseña incorrectos','error');
      usuarioActual = data;
      render();
    }

    function logout() {
      usuarioActual = null;
      render();
    }

    // Cargar colores
    async function cargarColores() {
      const { data } = await supabase.from('colores').select('*');
      colores = data||[];
    }

    // Cargar embolses
    async function cargarEmbolses() {
      const { data } = await supabase.from('embolses').select('*').order('fecha', {ascending:false});
      embolses = data||[];
    }

    // Cargar usuarios (admin)
    async function cargarUsuarios() {
      const { data } = await supabase.from('usuarios').select('*');
      usuarios = data||[];
    }

    // Registrar embolse
    async function registrarEmbolse() {
      const parcela = document.getElementById('parcela').value.trim();
      const cinta = document.getElementById('cinta').value;
      const color = colores.find(c=>c.nombre===cinta);
      const cantidad = parseInt(document.getElementById('cantidad').value,10);
      const fecha = document.getElementById('fecha').value;
      if (!parcela || !cinta || !color || !cantidad || !fecha) return showMsg('Completa todos los campos','error');
      const { error } = await supabase.from('embolses').insert([{
        parcela, cinta, color_hex: color.hex, cantidad, fecha,
        usuario_id: usuarioActual.id, usuario_nombre: usuarioActual.nombre
      }]);
      if (error) return showMsg('Error al guardar','error');
      showMsg('Registro guardado');
      render();
    }

    // Eliminar embolse
    async function eliminarEmbolse(id) {
      if (!confirm('¿Eliminar este registro?')) return;
      await supabase.from('embolses').delete().eq('id', id);
      showMsg('Registro eliminado');
      render();
    }

    // Descargar CSV
    function descargarCSV() {
      let csv = 'Fecha,Parcela,Cinta,Cantidad,Registrado por\n';
      embolses.forEach(e=>{
        csv += `"${e.fecha}","${e.parcela}","${e.cinta}","${e.cantidad}","${e.usuario_nombre||''}"\n`;
      });
      const blob = new Blob([csv], {type:'text/csv'});
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      a.download = 'embolses.csv';
      a.click();
      URL.revokeObjectURL(url);
    }

    // Agregar color
    async function agregarColor(e) {
      e.preventDefault();
      const nombre = document.getElementById('color-nombre').value.trim();
      const hex = document.getElementById('color-hex').value;
      if (!nombre || !hex) return showMsg('Completa nombre y color','error');
      await supabase.from('colores').insert([{nombre, hex}]);
      showMsg('Color agregado');
      render();
    }

    // Eliminar color
    async function eliminarColor(id) {
      await supabase.from('colores').delete().eq('id', id);
      showMsg('Color eliminado');
      render();
    }

    // Agregar usuario
    async function agregarUsuario(e) {
      e.preventDefault();
      const nombre = document.getElementById('user-nombre').value.trim();
      const username = document.getElementById('user-username').value.trim();
      const password = document.getElementById('user-pass').value.trim();
      const rol = document.getElementById('user-rol').value;
      if (!nombre || !username || !password || !rol) return showMsg('Completa todos los campos','error');
      await supabase.from('usuarios').insert([{nombre, username, password, rol}]);
      showMsg('Usuario agregado');
      render();
    }

    // Eliminar usuario
    async function eliminarUsuario(id) {
      if (!confirm('¿Eliminar este usuario?')) return;
      await supabase.from('usuarios').delete().eq('id', id);
      showMsg('Usuario eliminado');
      render();
    }

    // Inicializa
    render();
  </script>
</body>
</html>
