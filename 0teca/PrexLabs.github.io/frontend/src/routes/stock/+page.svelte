<script>
  import { onMount } from 'svelte';

  let productos = [];
  let busqueda = '';
  let mostrarFormulario = false;
  let mensaje = '';

  let producto = {
    nombre: '',
    lote: '',
    vencimiento: '',
    cantidad: 1,
    proveedor: '',
    descripcion: ''
  };

  let foto = '';

  onMount(() => {
    const guardados = localStorage.getItem('stock_productos');
    if (guardados) {
      productos = JSON.parse(guardados);
    }
  });

  function guardarStock() {
    localStorage.setItem('stock_productos', JSON.stringify(productos));
  }

  function limpiarFormulario() {
    producto = {
      nombre: '',
      lote: '',
      vencimiento: '',
      cantidad: 1,
      proveedor: '',
      descripcion: ''
    };
    foto = '';
  }

  function agregarProducto() {
    if (!producto.nombre.trim()) {
      mensaje = '⚠️ Ingresá el nombre del producto.';
      return;
    }

    const nuevo = {
      id: Date.now(),
      ...producto,
      cantidad: Number(producto.cantidad) || 0,
      fechaAlta: new Date().toISOString(),
      foto
    };

    productos = [nuevo, ...productos];
    guardarStock();

    mensaje = '✅ Producto agregado correctamente.';
    limpiarFormulario();
    mostrarFormulario = false;

    setTimeout(() => {
      mensaje = '';
    }, 3000);
  }

  function eliminarProducto(id) {
    if (!confirm('¿Eliminar este producto del stock?')) return;

    productos = productos.filter((p) => p.id !== id);
    guardarStock();
  }

  function modificarCantidad(id, cambio) {
    productos = productos.map((p) => {
      if (p.id === id) {
        return {
          ...p,
          cantidad: Math.max(0, Number(p.cantidad) + cambio)
        };
      }

      return p;
    });

    guardarStock();
  }

  function manejarFoto(event) {
    const archivo = event.target.files?.[0];

    if (!archivo) return;

    const lector = new FileReader();

    lector.onload = (e) => {
      foto = e.target.result;
    };

    lector.readAsDataURL(archivo);
  }

  function diasParaVencer(fecha) {
    if (!fecha) return null;

    const hoy = new Date();
    hoy.setHours(0, 0, 0, 0);

    const vencimiento = new Date(fecha + 'T00:00:00');

    return Math.ceil(
      (vencimiento - hoy) / (1000 * 60 * 60 * 24)
    );
  }

  function estadoVencimiento(fecha) {
    const dias = diasParaVencer(fecha);

    if (dias === null) return 'sin-fecha';
    if (dias < 0) return 'vencido';
    if (dias <= 30) return 'urgente';
    if (dias <= 90) return 'proximo';

    return 'normal';
  }

  $: productosFiltrados = productos.filter((p) => {
    const texto = busqueda.toLowerCase();

    return (
      p.nombre?.toLowerCase().includes(texto) ||
      p.lote?.toLowerCase().includes(texto) ||
      p.proveedor?.toLowerCase().includes(texto) ||
      p.descripcion?.toLowerCase().includes(texto)
    );
  });

  $: totalProductos = productos.length;

  $: totalUnidades = productos.reduce(
    (total, p) => total + Number(p.cantidad || 0),
    0
  );

  $: productosVencidos = productos.filter(
    (p) => estadoVencimiento(p.vencimiento) === 'vencido'
  ).length;

  $: productosProximos = productos.filter((p) => {
    const estado = estadoVencimiento(p.vencimiento);
    return estado === 'urgente' || estado === 'proximo';
  }).length;
</script>

<svelte:head>
  <title>Stock</title>
  <meta
    name="viewport"
    content="width=device-width, initial-scale=1"
  />
</svelte:head>

<div class="pagina">

  <header class="encabezado">
    <div>
      <div class="titulo">📦 Control de Stock</div>
      <div class="subtitulo">
        Gestión de productos y vencimientos
      </div>
    </div>

    <button
      class="boton-principal"
      on:click={() => {
        mostrarFormulario = !mostrarFormulario;
        mensaje = '';
      }}
    >
      {mostrarFormulario ? '✕ Cerrar' : '＋ Producto'}
    </button>
  </header>

  {#if mensaje}
    <div class="mensaje">
      {mensaje}
    </div>
  {/if}

  <section class="resumen">

    <div class="tarjeta">
      <span class="icono">📦</span>
      <div>
        <strong>{totalProductos}</strong>
        <span>Productos</span>
      </div>
    </div>

    <div class="tarjeta">
      <span class="icono">🔢</span>
      <div>
        <strong>{totalUnidades}</strong>
        <span>Unidades</span>
      </div>
    </div>

    <div class="tarjeta alerta">
      <span class="icono">⏰</span>
      <div>
        <strong>{productosProximos}</strong>
        <span>Próximos a vencer</span>
      </div>
    </div>

    <div class="tarjeta peligro">
      <span class="icono">⚠️</span>
      <div>
        <strong>{productosVencidos}</strong>
        <span>Vencidos</span>
      </div>
    </div>

  </section>

  {#if mostrarFormulario}

    <section class="formulario">

      <h2>Agregar producto</h2>

      <div class="foto-area">

        {#if foto}
          <img
            src={foto}
            alt="Producto"
            class="vista-foto"
          />
        {:else}
          <div class="camara-icono">📷</div>
        {/if}

        <label class="boton-foto">
          📷 Sacar / subir foto
          <input
            type="file"
            accept="image/*"
            capture="environment"
            on:change={manejarFoto}
            hidden
          />
        </label>

        <small>
          La foto queda asociada al producto.
        </small>

      </div>

      <div class="campos">

        <div class="campo campo-grande">
          <label>Nombre del producto *</label>
          <input
            bind:value={producto.nombre}
            placeholder="Ej: Paracetamol 500 mg"
          />
        </div>

        <div class="campo">
          <label>Lote</label>
          <input
            bind:value={producto.lote}
            placeholder="Ej: ABC123"
          />
        </div>

        <div class="campo">
          <label>Vencimiento</label>
          <input
            type="date"
            bind:value={producto.vencimiento}
          />
        </div>

        <div class="campo">
          <label>Cantidad</label>
          <input
            type="number"
            min="0"
            bind:value={producto.cantidad}
          />
        </div>

        <div class="campo">
          <label>Proveedor</label>
          <input
            bind:value={producto.proveedor}
            placeholder="Nombre del proveedor"
          />
        </div>

        <div class="campo campo-grande">
          <label>Descripción / información adicional</label>
          <textarea
            bind:value={producto.descripcion}
            placeholder="Marca, presentación, ubicación, observaciones..."
          ></textarea>
        </div>

      </div>

      <div class="acciones">

        <button
          class="boton-secundario"
          on:click={limpiarFormulario}
        >
          Limpiar
        </button>

        <button
          class="boton-guardar"
          on:click={agregarProducto}
        >
          💾 Guardar producto
        </button>

      </div>

    </section>

  {/if}

  <section class="lista">

    <div class="barra-busqueda">
      <span>🔎</span>

      <input
        bind:value={busqueda}
        placeholder="Buscar por nombre, lote, proveedor o descripción..."
      />
    </div>

    {#if productosFiltrados.length === 0}

      <div class="vacio">
        <div>📦</div>
        <h3>No hay productos</h3>
        <p>
          Agregá el primer producto utilizando el botón
          «＋ Producto».
        </p>
      </div>

    {:else}

      <div class="productos">

        {#each productosFiltrados as p}

          <article class="producto">

            <div class="producto-foto">

              {#if p.foto}
                <img
                  src={p.foto}
                  alt={p.nombre}
                />
              {:else}
                <span>📦</span>
              {/if}

            </div>

            <div class="producto-info">

              <div class="producto-titulo">
                {p.nombre}
              </div>

              {#if p.descripcion}
                <div class="descripcion">
                  {p.descripcion}
                </div>
              {/if}

              <div class="datos">

                {#if p.lote}
                  <span>
                    🏷️ Lote: <b>{p.lote}</b>
                  </span>
                {/if}

                {#if p.proveedor}
                  <span>
                    🏭 {p.proveedor}
                  </span>
                {/if}

              </div>

              {#if p.vencimiento}

                <div
                  class="vencimiento {estadoVencimiento(p.vencimiento)}"
                >
                  {#if estadoVencimiento(p.vencimiento) === 'vencido'}
                    🔴 VENCIDO
                  {:else if estadoVencimiento(p.vencimiento) === 'urgente'}
                    🔴 Vence en {diasParaVencer(p.vencimiento)} días
                  {:else if estadoVencimiento(p.vencimiento) === 'proximo'}
                    🟠 Vence en {diasParaVencer(p.vencimiento)} días
                  {:else}
                    🟢 Vencimiento:
                  {/if}

                  {p.vencimiento}
                </div>

              {/if}

            </div>

            <div class="cantidad">

              <span>Cantidad</span>

              <div class="controles">

                <button
                  on:click={() => modificarCantidad(p.id, -1)}
                >
                  −
                </button>

                <strong>{p.cantidad}</strong>

                <button
                  on:click={() => modificarCantidad(p.id, 1)}
                >
                  ＋
                </button>

              </div>

              <button
                class="eliminar"
                on:click={() => eliminarProducto(p.id)}
              >
                🗑️
              </button>

            </div>

          </article>

        {/each}

      </div>

    {/if}

  </section>

</div>

<style>
  :global(body) {
    margin: 0;
    font-family:
      -apple-system,
      BlinkMacSystemFont,
      "Segoe UI",
      Roboto,
      sans-serif;
    background: #f4f6f8;
    color: #17202a;
  }

  .pagina {
    max-width: 1100px;
    margin: auto;
    padding: 20px;
  }

  .encabezado {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 15px;
    margin-bottom: 20px;
  }

  .titulo {
    font-size: 28px;
    font-weight: 800;
  }

  .subtitulo {
    color: #667085;
    margin-top: 5px;
  }

  button {
    border: 0;
    cursor: pointer;
    font-family: inherit;
  }

  .boton-principal {
    background: #2563eb;
    color: white;
    padding: 12px 18px;
    border-radius: 10px;
    font-size: 15px;
    font-weight: 700;
  }

  .mensaje {
    background: #e8f7ee;
    color: #16753b;
    padding: 12px 15px;
    border-radius: 10px;
    margin-bottom: 15px;
    font-weight: 600;
  }

  .resumen {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 12px;
    margin-bottom: 20px;
  }

  .tarjeta {
    background: white;
    border-radius: 14px;
    padding: 16px;
    display: flex;
    align-items: center;
    gap: 12px;
    box-shadow: 0 2px 8px rgba(0,0,0,.05);
  }

  .tarjeta strong {
    display: block;
    font-size: 23px;
  }

  .tarjeta span:last-child {
    display: block;
    color: #667085;
    font-size: 13px;
  }

  .icono {
    font-size: 27px;
  }

  .formulario {
    background: white;
    border-radius: 16px;
    padding: 20px;
    margin-bottom: 20px;
    box-shadow: 0 2px 10px rgba(0,0,0,.06);
  }

  .formulario h2 {
    margin-top: 0;
  }

  .foto-area {
    border: 2px dashed #cbd5e1;
    border-radius: 14px;
    padding: 20px;
    text-align: center;
    margin-bottom: 20px;
  }

  .camara-icono {
    font-size: 45px;
    margin-bottom: 10px;
  }

  .boton-foto {
    display: inline-block;
    background: #111827;
    color: white;
    padding: 11px 18px;
    border-radius: 9px;
    cursor: pointer;
    font-weight: 600;
  }

  .foto-area small {
    display: block;
    margin-top: 8px;
    color: #667085;
  }

  .vista-foto {
    width: 140px;
    height: 140px;
    object-fit: cover;
    border-radius: 12px;
    display: block;
    margin: 0 auto 12px;
  }

  .campos {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 15px;
  }

  .campo {
    display: flex;
    flex-direction: column;
    gap: 7px;
  }

  .campo-grande {
    grid-column: span 2;
  }

  .campo label {
    font-weight: 700;
    font-size: 14px;
  }

  input,
  textarea {
    width: 100%;
    box-sizing: border-box;
    border: 1px solid #d0d5dd;
    border-radius: 9px;
    padding: 12px;
    font-size: 15px;
    background: white;
  }

  textarea {
    min-height: 90px;
    resize: vertical;
  }

  .acciones {
    display: flex;
    justify-content: flex-end;
    gap: 10px;
    margin-top: 20px;
  }

  .boton-secundario {
    background: #eef2f6;
    padding: 12px 18px;
    border-radius: 9px;
  }

  .boton-guardar {
    background: #16a34a;
    color: white;
    padding: 12px 20px;
    border-radius: 9px;
    font-weight: 700;
  }

  .barra-busqueda {
    background: white;
    border-radius: 12px;
    padding: 5px 14px;
    display: flex;
    align-items: center;
    gap: 10px;
    margin-bottom: 15px;
    box-shadow: 0 2px 8px rgba(0,0,0,.05);
  }

  .barra-busqueda input {
    border: 0;
    outline: none;
  }

  .producto {
    background: white;
    border-radius: 15px;
    padding: 15px;
    margin-bottom: 12px;
    display: grid;
    grid-template-columns: 75px 1fr auto;
    gap: 15px;
    align-items: center;
    box-shadow: 0 2px 8px rgba(0,0,0,.05);
  }

  .producto-foto {
    width: 75px;
    height: 75px;
    border-radius: 12px;
    background: #f1f5f9;
    display: flex;
    align-items: center;
    justify-content: center;
    overflow: hidden;
    font-size: 32px;
  }

  .producto-foto img {
    width: 100%;
    height: 100%;
    object-fit: cover;
  }

  .producto-titulo {
    font-size: 18px;
    font-weight: 800;
  }

  .descripcion {
    color: #667085;
    margin: 4px 0 8px;
    font-size: 13px;
  }

  .datos {
    display: flex;
    flex-wrap: wrap;
    gap: 12px;
    color: #475467;
    font-size: 13px;
  }

  .vencimiento {
    margin-top: 8px;
    font-size: 13px;
    font-weight: 700;
  }

  .vencimiento.normal {
    color: #15803d;
  }

  .vencimiento.proximo {
    color: #d97706;
  }

  .vencimiento.urgente,
  .vencimiento.vencido {
    color: #dc2626;
  }

  .cantidad {
    text-align: center;
  }

  .cantidad > span {
    display: block;
    font-size: 12px;
    color: #667085;
    margin-bottom: 5px;
  }

  .controles {
    display: flex;
    align-items: center;
    gap: 8px;
  }

  .controles button {
    width: 32px;
    height: 32px;
    border-radius: 8px;
    background: #eef2f6;
    font-size: 18px;
  }

  .controles strong {
    min-width: 30px;
  }

  .eliminar {
    margin-top: 10px;
    background: transparent;
    font-size: 18px;
  }

  .vacio {
    background: white;
    border-radius: 15px;
    padding: 50px 20px;
    text-align: center;
    color: #667085;
  }

  .vacio div {
    font-size: 50px;
  }

  .vacio h3 {
    color: #17202a;
  }

  @media (max-width: 700px) {

    .pagina {
      padding: 12px;
    }

    .encabezado {
      align-items: flex-start;
    }

    .titulo {
      font-size: 23px;
    }

    .resumen {
      grid-template-columns: repeat(2, 1fr);
    }

    .campos {
      grid-template-columns: 1fr;
    }

    .campo-grande {
      grid-column: span 1;
    }

    .producto {
      grid-template-columns: 60px 1fr;
    }

    .producto-foto {
      width: 60px;
      height: 60px;
    }

    .cantidad {
      grid-column: 1 / -1;
      display: flex;
      justify-content: space-between;
      align-items: center;
    }

    .cantidad > span {
      margin: 0;
    }

    .eliminar {
      margin: 0;
    }
  }
</style>
