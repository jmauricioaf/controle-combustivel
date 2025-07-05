// ** ATENÇÃO: Substitua pelos seus valores do Supabase! **
const SUPABASE_URL = 'https://cauuxhfljaxnkdrbgkso.supabase.co'; // Ex: 'https://abcdefghijk.supabase.co'
const SUPABASE_ANON_KEY = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6ImNhdXV4aGZsamF4bmtkcmJna3NvIiwicm9sZSI6ImFub24iLCJpYXQiOjE3NTE3NDE4NTQsImV4cCI6MjA2NzMxNzg1NH0.R6LarbWyV6I51TiBRtm_ORRkyNdxwN1oVXx0ELMTwNE'; // Ex: 'eyJhbGciOiJIUzI1Ni...'


const supabase = Supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);

// Elementos da tela de login
const loginScreen = document.getElementById('loginScreen');
const loginForm = document.getElementById('loginForm');
const placaLoginInput = document.getElementById('placaLoginInput');
const loginMessage = document.getElementById('loginMessage');

// Elementos do conteúdo principal do aplicativo
const appContent = document.getElementById('appContent');
const currentVehiclePlate = document.getElementById('currentVehiclePlate'); // Para exibir a placa logada
const registroForm = document.getElementById('registroForm');
const placaInput = document.getElementById('placaInput'); // Hidden input para a placa
const valorLitroInput = document.getElementById('valorLitroInput');
const quantidadeLitrosInput = document.getElementById('quantidadeLitrosInput');
const precoTotalInput = document.getElementById('precoTotalInput');
const quilometragemInput = document.getElementById('quilometragemInput');
const tipoCombustivelSelect = document.getElementById('tipoCombustivelSelect');
const formMessage = document.getElementById('formMessage');

const registrosTableBody = document.getElementById('registrosTableBody');
const metricasContainer = document.getElementById('metricasContainer');
const placaExibida = document.getElementById('placaExibida');
const consumoMedio = document.getElementById('consumoMedio');
const totalAbastecimentos = document.getElementById('totalAbastecimentos');
const logoutBtn = document.getElementById('logoutBtn');

let currentPlate = ''; // Variável para armazenar a placa do veículo logado

// --- Funções de Controle de Tela ---
function showLoginScreen() {
    loginScreen.style.display = 'block';
    appContent.style.display = 'none';
    loginForm.reset();
    loginMessage.textContent = '';
    currentPlate = ''; // Limpa a placa ao sair
}

function showAppContent(placa) {
    loginScreen.style.display = 'none';
    appContent.style.display = 'block';
    currentPlate = placa.toUpperCase().trim();
    currentVehiclePlate.textContent = currentPlate; // Exibe a placa no título
    placaInput.value = currentPlate; // Preenche o hidden input da placa
    registroForm.reset(); // Limpa o formulário de registro
    formMessage.textContent = '';
    buscarRegistrosEAtualizar(currentPlate); // Carrega os dados para a placa logada
}

// --- Lógica de Login ---
loginForm.addEventListener('submit', (e) => {
    e.preventDefault();
    const placaDigitada = placaLoginInput.value.toUpperCase().trim();
    if (placaDigitada) {
        showAppContent(placaDigitada);
    } else {
        mostrarMensagemLogin('Por favor, digite a placa do veículo.', 'danger');
    }
});

logoutBtn.addEventListener('click', () => {
    showLoginScreen();
});

// Função para mostrar mensagens temporárias na tela de login
function mostrarMensagemLogin(mensagem, tipo) {
    loginMessage.textContent = mensagem;
    loginMessage.className = `mt-3 text-center alert alert-${tipo}`;
    setTimeout(() => {
        loginMessage.textContent = '';
        loginMessage.className = 'mt-3 text-center';
    }, 5000);
}

// --- Funções de Cálculo e Validação (Mantidas do código anterior) ---
function calcularCampoFaltante() {
    let valorLitro = parseFloat(valorLitroInput.value);
    let quantidadeLitros = parseFloat(quantidadeLitrosInput.value);
    let precoTotal = parseFloat(precoTotalInput.value);

    if (valorLitro && quantidadeLitros && !precoTotal) {
        precoTotalInput.value = (valorLitro * quantidadeLitros).toFixed(2);
    } else if (valorLitro && precoTotal && !quantidadeLitros) {
        quantidadeLitrosInput.value = (precoTotal / valorLitro).toFixed(2);
    } else if (quantidadeLitros && precoTotal && !valorLitro) {
        valorLitroInput.value = (precoTotal / quantidadeLitros).toFixed(2);
    }
}

valorLitroInput.addEventListener('input', calcularCampoFaltante);
quantidadeLitrosInput.addEventListener('input', calcularCampoFaltante);
precoTotalInput.addEventListener('input', calcularCampoFaltante);

// --- Função para Registrar Abastecimento (Atualizada para usar currentPlate) ---
registroForm.addEventListener('submit', async (e) => {
    e.preventDefault();

    // Validar que pelo menos 2 dos 3 campos de preço/quantidade estão preenchidos
    const preenchidos = [valorLitroInput.value, quantidadeLitrosInput.value, precoTotalInput.value].filter(Boolean).length;
    if (preenchidos < 2) {
        mostrarMensagem('Por favor, preencha pelo menos 2 dos 3 campos: Valor do Litro, Quantidade de Litros ou Preço Total.', 'danger');
        return;
    }

    const registro = {
        placa: currentPlate, // Usa a placa logada
        valor_litro: parseFloat(valorLitroInput.value) || null,
        quantidade_litros: parseFloat(quantidadeLitrosInput.value) || null,
        preco_total: parseFloat(precoTotalInput.value) || null,
        quilometragem: parseInt(quilometragemInput.value),
        tipo_combustivel: tipoCombustivelSelect.value
    };

    // Validação adicional para quilometragem
    if (isNaN(registro.quilometragem) || registro.quilometragem <= 0) {
        mostrarMensagem('Quilometragem deve ser um número válido e maior que zero.', 'danger');
        return;
    }

    try {
        const { data, error } = await supabase
            .from('registros_combustivel')
            .insert([registro]);

        if (error) {
            console.error('Erro ao registrar:', error.message);
            mostrarMensagem(`Erro ao registrar: ${error.message}`, 'danger');
        } else {
            console.log('Registro inserido com sucesso:', data);
            mostrarMensagem('Abastecimento registrado com sucesso!', 'success');
            registroForm.reset();
            // Não precisa mais preencher placaInput, pois ele é hidden e já tem currentPlate
            buscarRegistrosEAtualizar(currentPlate); // Atualiza os dados da placa atual
        }
    } catch (err) {
        console.error('Erro inesperado:', err.message);
        mostrarMensagem(`Erro inesperado: ${err.message}`, 'danger');
    }
});

// Função para mostrar mensagens temporárias no formulário de registro
function mostrarMensagem(mensagem, tipo) {
    formMessage.textContent = mensagem;
    formMessage.className = `mt-3 text-center alert alert-${tipo}`;
    setTimeout(() => {
        formMessage.textContent = '';
        formMessage.className = 'mt-3 text-center';
    }, 5000);
}

// --- Função para buscar e exibir registros e calcular métricas (Atualizada para usar currentPlate) ---
async function buscarRegistrosEAtualizar(placa) {
    registrosTableBody.innerHTML = '<tr><td colspan="7" class="text-center">Carregando...</td></tr>';
    metricasContainer.style.display = 'none';

    try {
        const { data, error } = await supabase
            .from('registros_combustivel')
            .select('*')
            .eq('placa', placa) // Usa a placa passada como argumento
            .order('quilometragem', { ascending: true }); // Ordena pela quilometragem para cálculo de KM/L

        if (error) {
            console.error('Erro ao buscar registros:', error.message);
            registrosTableBody.innerHTML = `<tr><td colspan="7" class="text-center text-danger">Erro ao carregar registros: ${error.message}</td></tr>`;
            return;
        }

        if (data.length === 0) {
            registrosTableBody.innerHTML = '<tr><td colspan="7" class="text-center">Nenhum registro encontrado para esta placa.</td></tr>';
            placaExibida.textContent = placa;
            consumoMedio.textContent = 'N/A';
            totalAbastecimentos.textContent = '0';
            return;
        }

        // Exibir Registros
        registrosTableBody.innerHTML = ''; // Limpa antes de adicionar novos
        data.forEach(registro => {
            const row = registrosTableBody.insertRow();
            row.insertCell().textContent = new Date(registro.data_registro).toLocaleDateString('pt-BR');
            row.insertCell().textContent = registro.valor_litro ? `R$ ${registro.valor_litro.toFixed(2)}` : 'N/A';
            row.insertCell().textContent = registro.quantidade_litros ? `${registro.quantidade_litros.toFixed(2)} L` : 'N/A';
            row.insertCell().textContent = registro.preco_total ? `R$ ${registro.preco_total.toFixed(2)}` : 'N/A';
            row.insertCell().textContent = `${registro.quilometragem} km`;
            row.insertCell().textContent = tipoCombustivelParaTexto(registro.tipo_combustivel);
            
            const acoesCell = row.insertCell();
            const deleteBtn = document.createElement('button');
            deleteBtn.textContent = 'Excluir';
            deleteBtn.classList.add('btn', 'btn-danger', 'btn-sm');
            deleteBtn.onclick = () => excluirRegistro(registro.id, placa); // Passa a placa atual
            acoesCell.appendChild(deleteBtn);
        });

        // Calcular Métricas
        calcularMetricas(data);
        placaExibida.textContent = placa;
        metricasContainer.style.display = 'block';

    } catch (err) {
        console.error('Erro inesperado ao buscar registros:', err.message);
        registrosTableBody.innerHTML = `<tr><td colspan="7" class="text-center text-danger">Erro inesperado ao carregar: ${err.message}</td></tr>`;
    }
}

// Função auxiliar para converter tipo de combustível para texto amigável
function tipoCombustivelParaTexto(valor) {
    switch (valor) {
        case 'gasolina_comum': return 'Gasolina Comum';
        case 'gasolina_aditivada': return 'Gasolina Aditivada';
        case 'etanol': return 'Etanol';
        default: return valor;
    }
}

// Função para calcular métricas (KM/L)
function calcularMetricas(registros) {
    if (registros.length < 2) {
        consumoMedio.textContent = 'Necessita de mais de um registro para calcular.';
        totalAbastecimentos.textContent = registros.length;
        return;
    }

    const primeiroRegistro = registros[0];
    const ultimoRegistro = registros[registros.length - 1];

    let litrosTotais = 0;
    for (let i = 1; i < registros.length; i++) { // Começa do segundo registro
        if (registros[i].quantidade_litros) {
            litrosTotais += parseFloat(registros[i].quantidade_litros);
        }
    }

    const kmPercorridos = ultimoRegistro.quilometragem - primeiroRegistro.quilometragem;

    if (litrosTotais > 0 && kmPercorridos > 0) {
        const mediaKmPorLitro = kmPercorridos / litrosTotais;
        consumoMedio.textContent = `${mediaKmPorLitro.toFixed(2)} km/l`;
    } else {
        consumoMedio.textContent = 'N/A (Dados insuficientes)';
    }
    totalAbastecimentos.textContent = registros.length;
}

// Função para excluir registro (Atualizada para usar currentPlate)
async function excluirRegistro(id, placaAtual) {
    if (!confirm('Tem certeza que deseja excluir este registro?')) {
        return;
    }

    try {
        const { error } = await supabase
            .from('registros_combustivel')
            .delete()
            .eq('id', id);

        if (error) {
            console.error('Erro ao excluir:', error.message);
            alert(`Erro ao excluir: ${error.message}`);
        } else {
            alert('Registro excluído com sucesso!');
            buscarRegistrosEAtualizar(placaAtual); // Recarrega os registros
        }
    } catch (err) {
        console.error('Erro inesperado ao excluir:', err.message);
        alert(`Erro inesperado ao excluir: ${err.message}`);
    }
}

// --- Inicialização: Mostra a tela de login ao carregar a página ---
document.addEventListener('DOMContentLoaded', showLoginScreen);
