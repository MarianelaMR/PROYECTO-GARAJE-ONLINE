document.addEventListener('DOMContentLoaded', () => {

    // --- Selectores del DOM ---
    const containers = {
        vehiculo: document.getElementById('vehiclesContainer'),
        pasola: document.getElementById('scootersContainer'),
        search: document.getElementById('searchResultsContainer')
    };
    const searchInput = document.getElementById('searchInput');
    const searchResultsSection = document.getElementById('searchResults');
    const allProductsSections = document.getElementById('allProductsSections');
    const loadingSpinner = document.getElementById('loadingSpinner');
    const cartCount = document.getElementById('cartCount');
    const cartItemsContainer = document.getElementById('cartItems');
    const cartTotalSpan = document.getElementById('cartTotal');
    const checkoutBtn = document.getElementById('checkoutBtn');
    const processPaymentBtn = document.getElementById('processPaymentBtn');
    
    // Selectores del Modal de Detalles
    const productDetailModalEl = document.getElementById('productDetailModal');
    const productDetailModal = new bootstrap.Modal(productDetailModalEl);
    const modalProductName = document.getElementById('modalProductName');
    const modalProductImage = document.getElementById('modalProductImage');
    const modalProductPrice = document.getElementById('modalProductPrice');
    const modalProductColors = document.getElementById('modalProductColors');
    const quantityInput = document.getElementById('quantityInput');
    const addToCartBtn = document.getElementById('addToCartBtn');
    const colorSelectorContainer = document.getElementById('colorSelectorContainer');

    // --- Estado de la Aplicación ---
    let allProducts = [];
    let cart = [];
    let currentProduct = null;

    // --- Modales de Bootstrap ---
    const cartModal = new bootstrap.Modal(document.getElementById('cartModal'));
    const paymentModal = new bootstrap.Modal(document.getElementById('paymentModal'));

    // --- Lógica Principal ---
    // *** CAMBIO CLAVE: Se utiliza tu URL de GitHub ***
    const DATA_URL = 'https://raw.githubusercontent.com/MarianelaMR/GARAGE-EN-LINEA/refs/heads/main/vehiculos.json';

    /**
     * Carga los productos desde el archivo JSON.
     */
    const loadProducts = async () => {
        loadingSpinner.style.display = 'flex';
        try {
            const response = await fetch(DATA_URL);
            if (!response.ok) throw new Error(`La URL del JSON no es accesible. Status: ${response.status}`);
            allProducts = await response.json();
            distributeProducts(allProducts);
        } catch (error) {
            console.error("Error al cargar los productos:", error);
            allProductsSections.innerHTML = `<div class="text-center text-danger">
                                                <h4 class="fw-bold">Error al Cargar el Catálogo</h4>
                                                <p>No se pudo obtener la información de los productos desde el repositorio.</p>
                                             </div>`;
        } finally {
            loadingSpinner.style.display = 'none';
        }
    };

    /**
     * Distribuye los productos en sus respectivas secciones.
     * @param {Array} products - El array completo de productos.
     */
    const distributeProducts = (products) => {
        const categorized = { vehiculo: [], pasola: [] };
        
        products.forEach(p => {
            if (categorized[p.categoria_tienda]) {
                categorized[p.categoria_tienda].push(p);
            }
        });
        
        displayProducts(containers.vehiculo, categorized.vehiculo);
        displayProducts(containers.pasola, categorized.pasola);
    };

    /**
     * Renderiza las tarjetas de producto en su contenedor.
     * @param {HTMLElement} container - El elemento del DOM donde se insertarán las tarjetas.
     * @param {Array} products - Array de productos a mostrar.
     */
    const displayProducts = (container, products) => {
        container.innerHTML = '';
        if (products.length === 0) {
            container.innerHTML = `<p class="text-center text-muted">No se encontraron productos.</p>`;
            return;
        }
        products.forEach(product => {
            const card = document.createElement('div');
            card.className = 'col-lg-4 col-md-6 mb-4';
            card.innerHTML = `
                <div class="card h-100">
                    <img src="${product.imagen}" class="card-img-top" alt="${product.marca} ${product.modelo}" loading="lazy">
                    <div class="card-body d-flex flex-column">
                        <h5 class="card-title">${product.marca} ${product.modelo}</h5>
                        <p class="card-text text-muted flex-grow-1">${product.categoria}</p>
                        <p class="h5 price-text fw-bold mt-auto">${product.precio_venta.toLocaleString('en-US', { style: 'currency', currency: 'USD' })}</p>
                    </div>
                    <div class="card-footer text-center">
                         <button class="btn btn-outline-dark w-100 view-details-btn" data-codigo="${product.codigo}">Ver Detalles</button>
                    </div>
                </div>
            `;
            container.appendChild(card);
        });
    };

    /**
     * Muestra el modal con los detalles de un producto específico.
     * @param {object} product - El objeto del producto a mostrar.
     */
    const showProductDetailModal = (product) => {
        currentProduct = product;
        modalProductName.textContent = `${product.marca} ${product.modelo}`;
        modalProductImage.src = product.imagen;
        modalProductPrice.textContent = product.precio_venta.toLocaleString('en-US', { style: 'currency', currency: 'USD' });
        
        modalProductColors.innerHTML = '';
        if (product.colores_disponibles && product.colores_disponibles.length > 0) {
            colorSelectorContainer.style.display = 'block';
            product.colores_disponibles.forEach((color, index) => {
                const swatch = document.createElement('div');
                swatch.className = 'color-swatch';
                swatch.style.backgroundColor = color;
                swatch.dataset.color = color;
                if (index === 0) swatch.classList.add('selected');
                
                swatch.onclick = () => {
                    modalProductColors.querySelector('.selected')?.classList.remove('selected');
                    swatch.classList.add('selected');
                };
                modalProductColors.appendChild(swatch);
            });
        } else {
            colorSelectorContainer.style.display = 'none';
        }
        
        quantityInput.value = '1';
        productDetailModal.show();
    };

    /**
     * Añade un producto al carrito de compras.
     */
    const addItemToCart = () => {
        if (!currentProduct) return;
        const quantity = parseInt(quantityInput.value);
        if (quantity <= 0) return;

        let selectedColor = null;
        if (colorSelectorContainer.style.display !== 'none') {
            const selectedSwatch = modalProductColors.querySelector('.selected');
            if(selectedSwatch) selectedColor = selectedSwatch.dataset.color;
        }

        const cartItemId = `${currentProduct.codigo}-${selectedColor}`;
        const existingItem = cart.find(item => item.cartId === cartItemId);

        if (existingItem) {
            existingItem.quantity += quantity;
        } else {
            cart.push({ ...currentProduct, quantity, color: selectedColor, cartId: cartItemId });
        }
        
        updateCartUI();
        productDetailModal.hide();
    };

    /**
     * Actualiza la interfaz del carrito (contador y modal).
     */
    const updateCartUI = () => {
        cartItemsContainer.innerHTML = '';
        let total = 0;
        let totalItems = 0;

        if (cart.length === 0) {
            cartItemsContainer.innerHTML = '<p class="text-center">Tu carrito está vacío.</p>';
        } else {
            cart.forEach(item => {
                const subtotal = item.precio_venta * item.quantity;
                total += subtotal;
                totalItems += item.quantity;
                const itemName = `${item.marca} ${item.modelo}`;
                const itemColor = item.color ? `<small class="text-muted d-block">Color: <span class="d-inline-block" style="width:12px; height:12px; background-color:${item.color}; border-radius:50%; border: 1px solid #ccc; vertical-align: middle;"></span></small>` : '';

                const itemElement = document.createElement('div');
                itemElement.className = 'd-flex justify-content-between align-items-center mb-3 p-2 border rounded';
                itemElement.innerHTML = `
                    <div class="d-flex align-items-center">
                        <img src="${item.imagen}" alt="${itemName}" style="width: 70px; height: 70px; object-fit: contain; border-radius: 5px; padding: 5px;">
                        <div class="ms-3">
                            <h6 class="mb-0">${itemName}</h6>
                            ${itemColor}
                            <small class="text-muted">Cantidad: ${item.quantity}</small>
                        </div>
                    </div>
                    <strong>${subtotal.toLocaleString('en-US', { style: 'currency', 'currency': 'USD'})}</strong>
                `;
                cartItemsContainer.appendChild(itemElement);
            });
        }
        
        cartTotalSpan.textContent = total.toLocaleString('en-US', { style: 'currency', 'currency': 'USD'});
        cartCount.textContent = totalItems;
        if (totalItems > 0) {
            cartCount.style.animation = 'none';
            cartCount.offsetHeight;
            cartCount.style.animation = 'pulse 2s 1';
        }
    };

    /**
     * Filtra productos en tiempo real según la búsqueda del usuario.
     */
    const handleSearch = () => {
        const query = searchInput.value.toLowerCase().trim();
        if (query) {
            const filteredProducts = allProducts.filter(p =>
                p.marca.toLowerCase().includes(query) ||
                p.modelo.toLowerCase().includes(query)
            );
            allProductsSections.style.display = 'none';
            searchResultsSection.style.display = 'block';
            displayProducts(containers.search, filteredProducts);
        } else {
            allProductsSections.style.display = 'block';
            searchResultsSection.style.display = 'none';
        }
    };
    
    /**
     * Genera la factura en formato PDF.
     */
    const generateInvoice = () => {
        const { jsPDF } = window.jspdf;
        const doc = new jsPDF();
        
        const clientName = document.getElementById('cardName').value;
        const clientNif = document.getElementById('cardNif').value;
        const clientAddress = document.getElementById('cardAddress').value;

        const invoiceNumber = new Date().getTime();
        const purchaseDate = new Date().toLocaleDateString('es-ES');

        doc.setFont('helvetica', 'bold');
        doc.setFontSize(36);
        doc.text("MR", 15, 25);
        
        doc.setFontSize(10);
        doc.setFont('helvetica', 'normal');
        doc.text("Mercedes Dealer S.L.", 130, 15);
        doc.text("Av. del Lujo, 1, 28001 Madrid", 130, 20);
        doc.text("NIF: B12345678", 130, 25);
        doc.text("Tel: 912 345 678", 130, 30);
        doc.text("contacto@mercedesdealer.es", 130, 35);

        doc.setFontSize(22);
        doc.setFont('helvetica', 'bold');
        doc.text("FACTURA", 15, 50);
        
        doc.setFontSize(10);
        doc.setFont('helvetica', 'normal');
        doc.text(`Nº de factura: ${invoiceNumber}`, 15, 58);
        doc.text(`Fecha: ${purchaseDate}`, 15, 63);

        doc.setDrawColor(200, 200, 200);
        doc.rect(110, 50, 85, 25);
        doc.setFont('helvetica', 'bold');
        doc.text("Datos del cliente:", 115, 56);
        doc.setFont('helvetica', 'normal');
        doc.text(clientName, 115, 62);
        doc.text(`NIF: ${clientNif}`, 115, 67);
        doc.text(clientAddress, 115, 72);

        let y = 90;
        doc.setFillColor(50, 50, 50);
        doc.rect(15, y - 6, 180, 8, 'F');
        doc.setFont('helvetica', 'bold');
        doc.setTextColor(255, 255, 255);
        doc.text("CONCEPTO", 20, y);
        doc.text("CANT.", 115, y);
        doc.text("PRECIO", 140, y);
        doc.text("IMPORTE", 185, y, { align: 'right' });

        y += 7;
        doc.setTextColor(0, 0, 0);
        doc.setFont('helvetica', 'normal');
        let subtotal = 0;
        cart.forEach(item => {
            const itemTotal = item.quantity * item.precio_venta;
            subtotal += itemTotal;
            let description = `${item.marca} ${item.modelo}`;
            if (item.color) description += ` (Color: ${item.color})`;
            
            doc.text(description, 20, y);
            doc.text(item.quantity.toString(), 120, y, { align: 'center' });
            doc.text(item.precio_venta.toLocaleString('es-ES', { minimumFractionDigits: 2 }), 142, y);
            doc.text(itemTotal.toLocaleString('en-US', { style: 'currency', currency: 'USD' }), 195, y, { align: 'right' });
            y += 8;
        });

        if (y > 220) doc.addPage();
        const taxRate = 0.21;
        const tax = subtotal * taxRate;
        const total = subtotal + tax;
        
        doc.setDrawColor(200, 200, 200);
        doc.rect(110, y + 5, 85, 25);
        y += 11;
        doc.setFont('helvetica', 'bold');
        doc.text("BASE IMPONIBLE", 115, y);
        doc.setFont('helvetica', 'normal');
        doc.text(subtotal.toLocaleString('en-US', { style: 'currency', currency: 'USD' }), 195, y, { align: 'right' });
        y += 7;
        doc.setFont('helvetica', 'bold');
        doc.text(`IVA (${taxRate * 100}%)`, 115, y);
        doc.setFont('helvetica', 'normal');
        doc.text(tax.toLocaleString('en-US', { style: 'currency', currency: 'USD' }), 195, y, { align: 'right' });
        y += 7;
        doc.setFont('helvetica', 'bold');
        doc.text("TOTAL FACTURA", 115, y);
        doc.text(total.toLocaleString('en-US', { style: 'currency', currency: 'USD' }), 195, y, { align: 'right' });
        
        y = 270;
        doc.setFont('helvetica', 'bold');
        doc.text("Forma de pago:", 15, y);
        doc.setFont('helvetica', 'normal');
        doc.text("Transferencia bancaria a la cuenta ES12 3456 7890 1234 5678 9012", 50, y);

        y = 285;
        doc.setFontSize(8);
        doc.setTextColor(150, 150, 150);
        doc.text("Mercedes Dealer S.L. - Inscrita en el Registro Mercantil de Madrid, Tomo 123, Folio 45, Hoja M-67890.", 105, y, { align: 'center' });

        doc.save(`factura-mercedes-dealer-${invoiceNumber}.pdf`);
    };

    /**
     * Procesa el pago, valida el formulario y genera la factura.
     */
    const handlePayment = () => {
        const name = document.getElementById('cardName').value;
        const nif = document.getElementById('cardNif').value;
        const address = document.getElementById('cardAddress').value;

        if (!name || !nif || !address) {
            alert('Por favor, complete todos los datos del cliente para generar la factura.');
            return;
        }

        if (cart.length === 0) {
            alert('Tu carrito está vacío.');
            return;
        }
        alert('¡Compra finalizada con éxito! Se generará su factura.');
        generateInvoice();
        
        cart = [];
        updateCartUI();
        
        document.getElementById('paymentForm').reset();
        paymentModal.hide();
        cartModal.hide();
    };
    
    // --- Asignación de Event Listeners ---
    searchInput.addEventListener('input', handleSearch);
    addToCartBtn.addEventListener('click', addItemToCart);
    checkoutBtn.addEventListener('click', () => {
        if(cart.length > 0) {
            cartModal.hide();
            paymentModal.show();
        } else {
            alert("El carrito está vacío.");
        }
    });
    processPaymentBtn.addEventListener('click', handlePayment);

    // *** DELEGACIÓN DE EVENTOS (Método robusto) ***
    document.addEventListener('click', (event) => {
        const detailsButton = event.target.closest('.view-details-btn');
        if (detailsButton) {
            const codigo = parseInt(detailsButton.dataset.codigo);
            const product = allProducts.find(p => p.codigo === codigo);
            if (product) {
                showProductDetailModal(product);
            }
        }
    });

    // --- Carga Inicial ---
    loadProducts();
});