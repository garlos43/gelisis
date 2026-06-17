// =================================================================// Parceiro de Programação: FUSÃO ESTÁVEL (V56.19 - CONFIRMAÇÃO SILENCIOSA)// Descrição: Base antiga + confirmação silenciosa e nova tentativa se a pausa não aplicar// =================================================================

(function() {

// --- INÍCIO: KILL SWITCH ---
try {
    const ids = ['pausa-script-container', 'pausa-script-config-container', 'pausa-script-drag-overlay'];
    ids.forEach(id => { const el = document.getElementById(id); if(el) el.remove(); });
} catch (e) { console.warn(e); }
// --- FIM: KILL SWITCH ---

// ----------------------------------------------------
// CONSTANTES E CONFIGURAÇÕES
// ----------------------------------------------------
const SELECTORES = {
    MENU_STATUS_TOGGLE: '#user-settings-button',
    MENU_PRESENCA: '#presence-button',

    DISPONIVEL: 'input[value="Disponível"]',
    REFEICAO: 'input[value="Refeição"]',
    TREINAMENTO: 'input[value="Treinamento"]',
    REUNIAO: 'input[value="Reunião"]',

    INTERVALO_MENU: 'input[value="Intervalo"]',
    OCUPADO_MENU: 'input[value="Ocupado"]',

    LOGOFF: '[data-testid="user-setting-logout-button"]',

    STATUS_LABEL_TEXT: '.menu-row-text-label, .presence-label, gux-truncate',

    NA_FILA_DIRETO: 'input[value="Em fila"]',

    VOLTAR_SUBMENU: 'button[aria-label*="voltar"], button[title*="Voltar"]'
};

const PAUSAS_AGENDADAS_DEFAULT = [
    { hora: "10:40", status: "Na fila" }, { hora: "11:55", status: "Pausa out" },
    { hora: "12:00", status: "Descanso" }, { hora: "12:10", status: "Na fila" },
    { hora: "13:25", status: "Pausa out" }, { hora: "13:30", status: "Refeição" },
    { hora: "13:50", status: "Na fila" }, { hora: "15:15", status: "Pausa out" },
    { hora: "15:20", status: "Descanso" }, { hora: "15:30", status: "Na fila" },
    { hora: "16:55", status: "Pausa out" }, { hora: "17:00", status: "Logoff" }
];

// --- Variáveis de Controle ---
let estadoAtual = { tipo: 'Disponível', inicio: new Date(), ativa: false };
let isAgendamentoAtivo = false;
let schedulerInterval = null;
let ultimoEventoProcessado = null;
let isConfigPainelAberto = false;
let isAutomacaoPausada = false;
let globalWasDragging = false; 

// --- CONFIRMAÇÃO SILENCIOSA DE PAUSA ---
let confirmacaoAgendamentoAtual = null;
const MAX_TENTATIVAS_CONFIRMACAO = 3;
const DELAY_CONFIRMAR_STATUS = 3500;

// --- VARIÁVEL NOVA PARA CORREÇÃO DA EDIÇÃO ---
let indiceEdicao = -1; // -1 significa que não está editando, >= 0 é o índice sendo editado

// --- Chaves de Armazenamento ---
const STORAGE_KEY_HORARIOS = 'gerenciadorPausas_horarios';
const STORAGE_KEY_HISTORICO = 'gerenciadorPausas_historicoLog';
const STORAGE_KEY_POS_MAIN = 'gerenciadorPausas_posMain';
const STORAGE_KEY_POS_CONFIG = 'gerenciadorPausas_posConfig';
const STORAGE_KEY_NOTIF_CONFIG = 'gerenciadorPausas_notifConfig';
const STORAGE_KEY_ESTADO_ATUAL = 'gerenciadorPausas_estadoAtual_v2'; 
const STORAGE_KEY_AUTOMACAO_ATIVA = 'gerenciadorPausas_automacaoAtiva';

const NOTIF_CONFIG_DEFAULT = { ativadas: true, somAtivado: false, antecedenciaSegundos: 5 };
let configNotificacao = { ...NOTIF_CONFIG_DEFAULT };
const SOM_NOTIFICACAO_URL = 'https://www.soundjay.com/button/sounds/beep-07a.mp3';
let audioNotificacao = new Audio(SOM_NOTIFICACAO_URL);
let audioDesbloqueado = false;

// --- Funções de LocalStorage ---
function salvarHorariosLocalStorage(h){try{localStorage.setItem(STORAGE_KEY_HORARIOS,JSON.stringify(h))}catch(e){}}
function carregarHorariosLocalStorage(){try{const h=localStorage.getItem(STORAGE_KEY_HORARIOS);if(h){const p=JSON.parse(h);if(Array.isArray(p))return p;}}catch(e){}return[...PAUSAS_AGENDADAS_DEFAULT]}

function salvarHistoricoLocalStorage(h){try{localStorage.setItem(STORAGE_KEY_HISTORICO,JSON.stringify(h))}catch(e){}}
function carregarHistoricoLocalStorage(){try{const h=localStorage.getItem(STORAGE_KEY_HISTORICO);if(h){const p=JSON.parse(h);if(Array.isArray(p))return p;}}catch(e){}return []}

function salvarPosicaoPainelLocalStorage(k,el){if(!el)return;try{let p={};if(el.style.top&&el.style.left){p={top:el.style.top,left:el.style.left}}else{const r=el.getBoundingClientRect();p={bottom:'30px',right:'30px'}}localStorage.setItem(k,JSON.stringify(p))}catch(e){}}
function carregarPosicaoPainelLocalStorage(k){try{const pS=localStorage.getItem(k);if(pS)return JSON.parse(pS)}catch(e){}return null}
function salvarConfiguracoesNotificacaoLocalStorage(c){try{localStorage.setItem(STORAGE_KEY_NOTIF_CONFIG,JSON.stringify(c))}catch(e){}}
function carregarConfiguracoesNotificacaoLocalStorage(){try{const cS=localStorage.getItem(STORAGE_KEY_NOTIF_CONFIG);if(cS)return{...NOTIF_CONFIG_DEFAULT,...JSON.parse(cS)}}catch(e){}return{...NOTIF_CONFIG_DEFAULT}}

// --- Novas Funções de Persistência de Estado ---
function salvarEstadoAtualLocalStorage() {
    try {
        const dataSalvar = {
            tipo: estadoAtual.tipo,
            inicioMs: estadoAtual.inicio.getTime(),
            ativa: estadoAtual.ativa
        };
        localStorage.setItem(STORAGE_KEY_ESTADO_ATUAL, JSON.stringify(dataSalvar));
    } catch(e) {}
}

function salvarStatusAutomacaoLocalStorage(ativo) {
    try { localStorage.setItem(STORAGE_KEY_AUTOMACAO_ATIVA, JSON.stringify(ativo)); } catch(e){}
}

function recuperarEstadoInicial() {
    // 1. Recuperar Automação
    try {
        const autoSalva = localStorage.getItem(STORAGE_KEY_AUTOMACAO_ATIVA);
        if (autoSalva) {
            const isAtivo = JSON.parse(autoSalva);
            if (isAtivo) {
                toggleAgendamento(true); 
            }
        }
    } catch(e) {}

    // 2. Recuperar Status Timer
    try {
        const estadoSalvo = localStorage.getItem(STORAGE_KEY_ESTADO_ATUAL);
        if (estadoSalvo) {
            const dados = JSON.parse(estadoSalvo);
            const agora = new Date().getTime();
            if (agora - dados.inicioMs < 43200000) { 
                estadoAtual = {
                    tipo: dados.tipo,
                    inicio: new Date(dados.inicioMs),
                    ativa: dados.ativa
                };
            }
        }
    } catch(e) {}
}

let PAUSAS_AGENDADAS = carregarHorariosLocalStorage();
PAUSAS_AGENDADAS = PAUSAS_AGENDADAS.map(p => ({
    ...p,
    status: normalizarStatusTexto(p.status) === 'pausa auricular' ? 'Descanso' : p.status
}));
salvarHorariosLocalStorage(PAUSAS_AGENDADAS);
let historicoPausas = carregarHistoricoLocalStorage(); 
configNotificacao = carregarConfiguracoesNotificacaoLocalStorage();

// ----------------------------------------------------
// FUNÇÕES DE UTILIDADE E INTERAÇÃO
// ----------------------------------------------------
function tocarSomNotificacao(){if(!configNotificacao.somAtivado)return;if(!audioDesbloqueado){desbloquearAudio(true);return}audioNotificacao.currentTime=0;audioNotificacao.play().catch(e=>{})}
function desbloquearAudio(forcePlay=!1){if(audioDesbloqueado&&!forcePlay)return;let p=audioNotificacao.play();if(p!==undefined){p.then(()=>{audioNotificacao.pause();audioNotificacao.currentTime=0;if(!audioDesbloqueado){audioDesbloqueado=!0}}).catch(e=>{});}if(audioDesbloqueado){document.removeEventListener('click',desbloquearAudio)}}
async function mostrarNotificacao(evento){if(!configNotificacao.ativadas)return;if(!("Notification"in window))return;let p=Notification.permission;if(p==="granted"){tocarSomNotificacao();new Notification("Gerenciador",{body:`${evento.status} às ${evento.hora}`,tag:`gdp-${evento.hora}`})}else if(p==="default"){try{p=await Notification.requestPermission();if(p==="granted")mostrarNotificacao(evento)}catch(e){}}}

function clicarElemento(s, t = null) {
    let e = null;
    if (t) {
        const l = document.querySelectorAll(s); 
        for (const o of l) {
            if (o.textContent.trim().toLowerCase() === t.toLowerCase()) {
                e = o;
                while (e && e.parentElement && e.tagName !== 'BUTTON') e = e.parentElement;
                if (e && e.tagName === 'BUTTON') break; 
                e = null;
            }
        }
    } else { e = document.querySelector(s); }
    
    if (e) {
        try { e.click(); } catch(err) {}
        const c = new MouseEvent('click', { view: window, bubbles: true, cancelable: true });
        e.dispatchEvent(c);
        return true;
    }
    return false;
}

function clicarVoltarSubmenu(){if(clicarElemento(SELECTORES.VOLTAR_SUBMENU))return!0;return!1}
function formatarDuracao(s){const h=Math.floor(s/3600),m=Math.floor((s%3600)/60),c=Math.max(0,s%60);const p=n=>String(n).padStart(2,'0');return`${p(h)}:${p(m)}:${p(c)}`}
function converterDuracaoParaSegundos(d){if(!d||typeof d!=='string')return 0;const p=d.split(':');if(p.length!==3)return 0;const h=parseInt(p[0],10)||0,m=parseInt(p[1],10)||0,s=parseInt(p[2],10)||0;return(h*3600)+(m*60)+s}

function registrarPausa(n){
    const a=new Date;
    if(estadoAtual.ativa){
        const f=a; const d=f.getTime()-estadoAtual.inicio.getTime();
        if(d>500){ 
            historicoPausas.push({
                tipo:estadoAtual.tipo,
                inicio:estadoAtual.inicio.toLocaleTimeString('pt-BR',{hour:'2-digit',minute:'2-digit',second:'2-digit'}),
                fim:f.toLocaleTimeString('pt-BR',{hour:'2-digit',minute:'2-digit',second:'2-digit'}),
                duracao:formatarDuracao(Math.round(d/1000))
            });
            salvarHistoricoLocalStorage(historicoPausas); 
            renderizarHistoricoLog(); renderizarEstatisticas();
        }
    } 
    estadoAtual={ tipo:n, inicio:a, ativa: (n !== 'Logoff') };
    salvarEstadoAtualLocalStorage(); 
    atualizarUI();
}

function adicionarAtalho(){document.addEventListener('keydown',function(e){const c=e.ctrlKey||e.metaKey,s=e.shiftKey,o=e.key==='1';if(document.activeElement&&['INPUT','TEXTAREA'].includes(document.activeElement.tagName))return;if(c&&s&&o){e.preventDefault();toggleVisibilidade()}})}

// --- Agendamento ---
function mapearEexecutarAcao(s){
    const status = s.toLowerCase().trim();
    switch(status){
        case"na fila": iniciarNaFila(true); break;
        case"disponível": voltarDisponivel(true); break;
        case"pausa out": iniciarPausaOut(true); break;
        case"pausa auricular": 
        case"pausa pessoal":
        case"descanso": iniciarDescanso(true); break;
        case"refeição": iniciarRefeicao(true); break;
        case"treinamento": iniciarTreinamento(true); break;
        case"reunião": iniciarReuniao(true); break;
        case"logoff": realizarLogoff(true); break;
        case"atividade backoffice":
        case"ativ. backoffice":
        case"backoffice": iniciarBackoffice(true); break;
        case"feedback": iniciarFeedback(true); break;
        case"particular": iniciarParticular(true); break;
        case"questões de saúde":
        case"saúde":
        case"saude": iniciarQuestoesSaude(true); break;
        default: console.warn(`Status '${s}' desconhecido.`);
    }

}

function normalizarStatusTexto(txt) {
    return (txt || '')
        .normalize('NFD')
        .replace(/[\u0300-\u036f]/g, '')
        .replace(/\s+/g, ' ')
        .trim()
        .toLowerCase();
}

function statusEhEquivalente(statusDesejado, statusTela) {
    const desejado = normalizarStatusTexto(statusDesejado);
    const tela = normalizarStatusTexto(statusTela);

    if (!tela) return false;

    const equivalencias = {
        'na fila': ['na fila', 'on queue'],
        'disponivel': ['disponivel', 'available'],
        'refeicao': ['refeicao', 'meal'],
        'pausa out': ['pausa out'],
        'pausa auricular': ['pausa auricular', 'descanso', 'intervalo'],
        'pausa pessoal': ['pausa pessoal', 'descanso', 'intervalo'],
        'descanso': ['descanso', 'pausa auricular', 'intervalo'],
        'treinamento': ['treinamento', 'training'],
        'reuniao': ['reuniao', 'meeting'],
        'atividade backoffice': ['atividade backoffice', 'ativ. backoffice', 'backoffice'],
        'ativ. backoffice': ['atividade backoffice', 'ativ. backoffice', 'backoffice'],
        'backoffice': ['atividade backoffice', 'ativ. backoffice', 'backoffice'],
        'feedback': ['feedback'],
        'particular': ['particular'],
        'questoes de saude': ['questoes de saude', 'saude'],
        'saude': ['questoes de saude', 'saude']
    };

    const lista = equivalencias[desejado] || [desejado];
    return lista.some(item => tela.includes(item));
}

function obterStatusAtualDaTela() {
    const candidatos = [];

    try {
        const toggle = document.querySelector(SELECTORES.MENU_STATUS_TOGGLE);
        if (toggle) candidatos.push(toggle.textContent);
    } catch(e) {}

    try {
        const filaDireto = document.querySelector(SELECTORES.NA_FILA_DIRETO);
        if (filaDireto) candidatos.push(filaDireto.textContent);
    } catch(e) {}

    try {
        document.querySelectorAll(SELECTORES.STATUS_LABEL_TEXT).forEach(el => {
            if (el && el.offsetParent !== null) candidatos.push(el.textContent);
        });
    } catch(e) {}

    try {
        document.querySelectorAll('[class*="presence"], [class*="status"], [aria-label*="status"], [aria-label*="Status"]').forEach(el => {
            if (el && el.offsetParent !== null) candidatos.push(el.textContent || el.getAttribute('aria-label') || '');
        });
    } catch(e) {}

    return candidatos
        .map(t => (t || '').trim())
        .filter(Boolean)
        .join(' | ');
}

function confirmarPausaAplicada(statusDesejado) {
    if (statusDesejado.toLowerCase().trim() === 'logoff') return true;

    const statusTela = obterStatusAtualDaTela();
    const ok = statusEhEquivalente(statusDesejado, statusTela);

    console.log(`[GERENCIADOR] Confirmação: desejado="${statusDesejado}" tela="${statusTela}" ok=${ok}`);
    return ok;
}

function executarAcaoComConfirmacao(evento, horaEvento, tentativa = 1) {
    if (!evento || !isAgendamentoAtivo || isAutomacaoPausada) return;

    confirmacaoAgendamentoAtual = {
        hora: horaEvento,
        status: evento.status,
        tentativa
    };

    console.log(`[GERENCIADOR] Executando ${evento.status} (${horaEvento}) - tentativa ${tentativa}/${MAX_TENTATIVAS_CONFIRMACAO}`);
    mapearEexecutarAcao(evento.status);

    setTimeout(() => {
        if (!confirmacaoAgendamentoAtual) return;
        if (confirmacaoAgendamentoAtual.hora !== horaEvento) return;
        if (!isAgendamentoAtivo || isAutomacaoPausada) return;

        if (confirmarPausaAplicada(evento.status)) {
            ultimoEventoProcessado = horaEvento;
            confirmacaoAgendamentoAtual = null;

            if(evento.status.toLowerCase()==='logoff'){
                setTimeout(()=>{if(isAgendamentoAtivo)toggleAgendamento()},500)
            }
            return;
        }

        if (tentativa < MAX_TENTATIVAS_CONFIRMACAO) {
            setTimeout(() => executarAcaoComConfirmacao(evento, horaEvento, tentativa + 1), 1500);
        } else {
            console.warn(`[GERENCIADOR] Não confirmou "${evento.status}" após ${MAX_TENTATIVAS_CONFIRMACAO} tentativas. Sem aviso na tela.`);
            ultimoEventoProcessado = horaEvento;
            confirmacaoAgendamentoAtual = null;
        }
    }, DELAY_CONFIRMAR_STATUS);
}

function verificarEExecutarAgendamentos(){
    if(!isAgendamentoAtivo||isAutomacaoPausada) return; 
    
    const a=new Date();
    const h=`${String(a.getHours()).padStart(2,'0')}:${String(a.getMinutes()).padStart(2,'0')}`;
    const s=a.getSeconds(); 
    
    Object.keys(window).forEach(k=>{if(k.startsWith('notif_')){const e=k.split('_')[1];if(e<h){clearTimeout(window[k]);delete window[k];}}}); 
    
    const eA=PAUSAS_AGENDADAS.find(p=>p.hora===h);
    
    if(eA && ultimoEventoProcessado !== h){
        if(!confirmacaoAgendamentoAtual || confirmacaoAgendamentoAtual.hora !== h){
            executarAcaoComConfirmacao(eA, h, 1);
        }
        return;
    }
    
    if(!eA && ultimoEventoProcessado && ultimoEventoProcessado !== h) { 
        ultimoEventoProcessado = null; 
    } 
    
    const pM=new Date(a.getTime()+6e4); 
    const hP=`${String(pM.getHours()).padStart(2,'0')}:${String(pM.getMinutes()).padStart(2,'0')}`;
    const eP=PAUSAS_AGENDADAS.find(p=>p.hora===hP);
    
    if(eP){
        const antecedencia = configNotificacao.antecedenciaSegundos; 
        const sRestantes = 60 - s;
        
        if(sRestantes <= antecedencia + 2 && sRestantes >= antecedencia - 2) {
             const tId=`notif_${eP.hora}_${eP.status}`;
             if(!window[tId]){
                 mostrarNotificacao(eP);
                 window[tId] = true; 
             }
        }
    }
}

function iniciarSchedulerSincronizado(){
    if(!isAgendamentoAtivo) return;
    verificarEExecutarAgendamentos();
    schedulerInterval = setInterval(verificarEExecutarAgendamentos, 5000);
}

function toggleAgendamento(forceState = null){
    const b=document.getElementById('btn-toggle-agendamento'),p=document.getElementById('btn-pause-resume-agendamento');
    
    let novoEstado = !isAgendamentoAtivo;
    if (forceState !== null) novoEstado = forceState;

    if(!novoEstado){
        if(schedulerInterval) clearInterval(schedulerInterval);
        schedulerInterval=null;
        isAgendamentoAtivo=!1;
        isAutomacaoPausada=!1;
        if(b){b.textContent="Ligar Automação";b.style.background = 'linear-gradient(135deg, #23a745, #2dc24f)';}
        if(p)p.style.display='none';
    }else{
        PAUSAS_AGENDADAS=carregarHorariosLocalStorage().map(p => ({...p, status: normalizarStatusTexto(p.status) === 'pausa auricular' ? 'Descanso' : p.status}));
        salvarHorariosLocalStorage(PAUSAS_AGENDADAS);
        ultimoEventoProcessado=null;
        isAutomacaoPausada=!1;
        isAgendamentoAtivo=!0;
        if(b){b.textContent="Desligar Automação";b.style.background = 'linear-gradient(135deg, #dc3545, #e84a5f)';}
        if(p){
            p.textContent="Pausar ⏸️";p.style.background = 'linear-gradient(135deg, #ffc107, #ffd25a)';p.style.color = '#fff';p.style.textShadow = '0 1px 2px rgba(0,0,0,0.45)';p.style.display='inline-block';
        }
        iniciarSchedulerSincronizado(); 
    }
    salvarStatusAutomacaoLocalStorage(isAgendamentoAtivo); 
    atualizarUI();
}

function togglePausaAutomacao(){
    if(!isAgendamentoAtivo)return;
    isAutomacaoPausada=!isAutomacaoPausada;
    const b=document.getElementById('btn-pause-resume-agendamento');
    if(isAutomacaoPausada){
        if(b){b.textContent="Retomar ▶️";b.style.background = 'linear-gradient(135deg, #007FFF, #009cff)';b.style.color = '#fff';}
    }else{
        if(b){b.textContent="Pausar ⏸️";b.style.background = 'linear-gradient(135deg, #ffc107, #ffd25a)';b.style.color = '#fff';b.style.textShadow = '0 1px 2px rgba(0,0,0,0.45)';}
        verificarEExecutarAgendamentos();
    }
    atualizarUI();
}

function recalcularAgendamentosEmTempoReal() {
    Object.keys(window).forEach(k=>{
        if(k.startsWith('notif_')){ clearTimeout(window[k]); delete window[k]; }
    });
    ultimoEventoProcessado = null; 
}

// --- CORREÇÃO AQUI: Função Adicionar/Editar ---
function adicionarNovoHorario(){
    const iH=document.getElementById('novo-horario-hora'),iS=document.getElementById('novo-horario-status');
    const btn=document.getElementById('btn-adicionar-horario');
    if(!iH||!iS)return;
    const h=iH.value,s=iS.value.trim();
    if(!h||!s)return;

    // VERIFICA SE ESTÁ EM MODO DE EDIÇÃO
    if (indiceEdicao >= 0 && indiceEdicao < PAUSAS_AGENDADAS.length) {
        // Remove o item original que estava sendo editado (substituição)
        PAUSAS_AGENDADAS.splice(indiceEdicao, 1);
        // Reseta o modo de edição
        indiceEdicao = -1;
        if (btn) {
            btn.innerHTML = "+"; 
            btn.style.background = ""; 
            btn.style.color = "";
        }
    } else {
        // Comportamento normal de duplicação só se NÃO estiver editando
        if(PAUSAS_AGENDADAS.some(p=>p.hora===h)){
            if(!confirm(`Já existe ${h}. Substituir?`))return;
            PAUSAS_AGENDADAS=PAUSAS_AGENDADAS.filter(p=>p.hora!==h);
        }
    }
    
    PAUSAS_AGENDADAS.push({hora:h,status:s});
    PAUSAS_AGENDADAS.sort((a,b)=>a.hora.localeCompare(b.hora));
    salvarHorariosLocalStorage(PAUSAS_AGENDADAS);
    
    iH.value='';iS.value='';
    renderizarAgendamentos();
    recalcularAgendamentosEmTempoReal();

    if(btn && btn.textContent === "+") {
        const txtOriginal = btn.innerHTML;
        btn.innerHTML = "✓";
        btn.style.background = "#50fa7b"; 
        setTimeout(() => { 
            btn.innerHTML = txtOriginal; 
            btn.style.background = ""; 
        }, 1000);
    }
}

function removerHorario(idx){
    if(idx<0||idx>=PAUSAS_AGENDADAS.length)return;
    if(confirm("Remover este horário?")){
        PAUSAS_AGENDADAS.splice(idx,1);
        salvarHorariosLocalStorage(PAUSAS_AGENDADAS);
        renderizarAgendamentos();
        recalcularAgendamentosEmTempoReal();
        
        // Se remover o item que estava editando, cancela a edição
        if(indiceEdicao === idx) {
             indiceEdicao = -1;
             const btn=document.getElementById('btn-adicionar-horario');
             const iH=document.getElementById('novo-horario-hora'),iS=document.getElementById('novo-horario-status');
             if(btn) { btn.innerHTML = "+"; btn.style.background = ""; btn.style.color = ""; }
             if(iH) iH.value=''; if(iS) iS.value='';
        }
    }
}

// --- CORREÇÃO AQUI: Função Carregar Edição ---
function carregarHorarioParaEdicao(idx){
    if(idx<0||idx>=PAUSAS_AGENDADAS.length)return;
    const item=PAUSAS_AGENDADAS[idx];
    const iH=document.getElementById('novo-horario-hora'),iS=document.getElementById('novo-horario-status');
    const btn=document.getElementById('btn-adicionar-horario');
    
    if(!iH||!iS)return;
    iH.value=item.hora;
    iS.value=nomeExibicaoStatus(item.status);
    iS.focus();
    iS.select();
    
    // Define qual índice está sendo editado e muda visual do botão
    indiceEdicao = idx;
    if (btn) {
        btn.innerHTML = "💾"; // Ícone de salvar
        btn.style.background = "#ffc107"; // Amarelo alerta
        btn.style.color = "#fff";
        btn.style.textShadow = "0 1px 2px rgba(0,0,0,0.45)";
    }
}

function exportarConfiguracoes(){try{const c={pausas:carregarHorariosLocalStorage(),notificacoes:carregarConfiguracoesNotificacaoLocalStorage(),posicoes:{main:carregarPosicaoPainelLocalStorage(STORAGE_KEY_POS_MAIN),config:carregarPosicaoPainelLocalStorage(STORAGE_KEY_POS_CONFIG)}};const d="data:text/json;charset=utf-8,"+encodeURIComponent(JSON.stringify(c,null,2));const l=document.createElement('a');l.setAttribute('href',d);l.setAttribute('download','config_gerenciador_pausas.json');document.body.appendChild(l);l.click();l.remove();}catch(e){alert("Erro ao exportar.")}}
function importarConfiguracoes(){const i=document.getElementById('import-file-input');if(i)i.click()}
function handleArquivoImportado(event){const a=event.target.files[0];if(!a)return;const r=new FileReader();r.onload=function(e){try{const c=JSON.parse(e.target.result);if(c&&c.pausas){salvarHorariosLocalStorage(c.pausas);salvarConfiguracoesNotificacaoLocalStorage(c.notificacoes||NOTIF_CONFIG_DEFAULT);salvarPosicaoPainelLocalStorage(STORAGE_KEY_POS_MAIN,c.posicoes?.main);salvarPosicaoPainelLocalStorage(STORAGE_KEY_POS_CONFIG,c.posicoes?.config);alert("Importado com sucesso! Recarregue a página.");}}catch(err){alert("Arquivo inválido.")}};r.readAsText(a)}

const DELAY_ABRIR_MENU = 700; const DELAY_ABRIR_SUBMENU = 500; const DELAY_FECHAR_MENU = 200;

function verificarEVoltarDoSubmenu() {
    if (clicarElemento(SELECTORES.VOLTAR_SUBMENU)) return true;
    const botoes = document.querySelectorAll('button');
    for (const b of botoes) {
        if (b.offsetParent === null) continue;
        const txt = b.textContent.trim();
        if (txt.includes('Ocupado') && (txt.includes('<') || b.querySelector('gux-icon'))) {
            b.click(); return true;
        }
    }
    return false;
}

function executarLogicaMenuSimples(seletorBotao, nomePausa, isAutomatico = false) {

    document.querySelector('#user-settings-button')?.click();

    setTimeout(() => {

        document.querySelector('#presence-button')?.click();

        setTimeout(() => {

            const radio = document.querySelector(seletorBotao);

            if (radio) {

                radio.click();

                radio.checked = true;

                radio.dispatchEvent(
                    new Event('change', { bubbles: true })
                );

                radio.dispatchEvent(
                    new Event('input', { bubbles: true })
                );

            } else {
                console.warn('Status não encontrado:', seletorBotao);
            }

        }, 800);

    }, 800);
}

function executarLogicaNaFila(isAutomatico = false) {

    document.querySelector('#user-settings-button')?.click();

    setTimeout(() => {

        document.querySelector('#presence-button')?.click();

        setTimeout(() => {

            const radio =
                document.querySelector('input[value="Em fila"]');

            if (radio) {
                radio.click();
                radio.checked = true;
                radio.dispatchEvent(
                    new Event('change', { bubbles: true })
                );
            }

        }, 800);

    }, 800);
}

function executarLogicaRefeicao(isAutomatico = false){ executarLogicaMenuSimples(SELECTORES.REFEICAO, 'Refeição', isAutomatico); }
function executarLogicaDisponivel(isAutomatico = false){ executarLogicaMenuSimples(SELECTORES.DISPONIVEL, 'Disponível', isAutomatico); }
function executarLogicaTreinamento(isAutomatico = false){ executarLogicaMenuSimples(SELECTORES.TREINAMENTO, 'Treinamento', isAutomatico); }
function executarLogicaReuniao(isAutomatico = false){ executarLogicaMenuSimples(SELECTORES.REUNIAO, 'Reunião', isAutomatico); }
function executarLogicaDescanso(isAutomatico = false){ executarLogicaSubMenu("Intervalo", "Pausa auricular", "Pausa auricular", isAutomatico); }
function executarLogicaPausaOut(isAutomatico = false){ executarLogicaSubMenu("Ocupado", "Pausa out", "Pausa out", isAutomatico); }
function executarLogicaBackoffice(isAutomatico = false){ executarLogicaSubMenu("Ocupado", "Atividade backoffice", "Backoffice", isAutomatico); }
function executarLogicaFeedback(isAutomatico = false){ executarLogicaSubMenu("Ocupado", "Feedback", "Feedback", isAutomatico); }
function executarLogicaParticular(isAutomatico = false){ executarLogicaSubMenu("Ausente", "Particular", "Particular", isAutomatico); }
function executarLogicaQuestoesSaude(isAutomatico = false){ executarLogicaSubMenu("Ausente", "Questões de saúde", "Saúde", isAutomatico); }

function executarLogicaLogoff(isAutomatico = false) {
    if (!isAutomatico && !confirm("Fazer logoff? Histórico será zerado.")) return;
    if (clicarElemento(SELECTORES.MENU_STATUS_TOGGLE)) {
        setTimeout(() => {
            if (clicarElemento(SELECTORES.LOGOFF, "Fazer logoff")) {
                historicoPausas=[];
                salvarHistoricoLocalStorage([]); 
                estadoAtual={tipo:'Logoff',inicio:new Date,ativa:!1};
                salvarEstadoAtualLocalStorage(); 
                atualizarUI();renderizarHistoricoLog();renderizarEstatisticas();
                if(isAgendamentoAtivo) toggleAgendamento();
            } else {
                clicarElemento(SELECTORES.MENU_STATUS_TOGGLE);
            }
        }, DELAY_ABRIR_MENU);
    }
}

function iniciarNaFila(isAutomatico = false){ registrarPausa("Na fila"); executarLogicaNaFila(isAutomatico); }
function iniciarRefeicao(isAutomatico = false){ registrarPausa("Refeição"); executarLogicaRefeicao(isAutomatico); }
function voltarDisponivel(isAutomatico = false){ registrarPausa("Disponível"); executarLogicaDisponivel(isAutomatico); }
function iniciarTreinamento(isAutomatico = false){ registrarPausa("Treinamento"); executarLogicaTreinamento(isAutomatico); }
function iniciarReuniao(isAutomatico = false){ registrarPausa("Reunião"); executarLogicaReuniao(isAutomatico); }
function iniciarDescanso(isAutomatico = false){ registrarPausa("Descanso"); executarLogicaDescanso(isAutomatico); }
function iniciarPausaOut(isAutomatico = false){ registrarPausa("Pausa out"); executarLogicaPausaOut(isAutomatico); }
function iniciarBackoffice(isAutomatico = false){ registrarPausa("Backoffice"); executarLogicaBackoffice(isAutomatico); }
function iniciarFeedback(isAutomatico = false){ registrarPausa("Feedback"); executarLogicaFeedback(isAutomatico); }
function iniciarParticular(isAutomatico = false){ registrarPausa("Particular"); executarLogicaParticular(isAutomatico); }
function iniciarQuestoesSaude(isAutomatico = false){ registrarPausa("Saúde"); executarLogicaQuestoesSaude(isAutomatico); }
function realizarLogoff(isAutomatico = false){ executarLogicaLogoff(isAutomatico); }

function criarDragOverlay() {
    if(document.getElementById('pausa-script-drag-overlay')) return;
    const o = document.createElement('div');
    o.id = 'pausa-script-drag-overlay';
    o.style.cssText = 'position:fixed;top:0;left:0;width:100vw;height:100vh;z-index:999999;display:none;cursor:grabbing;';
    document.body.appendChild(o);
    return o;
}

function setupGPUDrag(elementId, headerId, storageKey) {
    const element = document.getElementById(elementId);
    const header = document.getElementById(headerId);
    const icone = document.getElementById('pausa-script-icone');
    const overlay = document.getElementById('pausa-script-drag-overlay');
    
    if (!element || !header || !overlay) return;

    let isDragging = false;
    let isRealMovement = false; 
    let startX, startY;
    let initialLeft, initialTop;
    let currentX, currentY; 
    let rAF = null; 
    
    function updatePosition() {
        if (!isDragging || !isRealMovement) return;
        element.style.transform = `translate3d(${currentX}px, ${currentY}px, 0)`;
        rAF = requestAnimationFrame(updatePosition);
    }

    function onPointerDown(e) {
        const isHeader = header.contains(e.target);
        const isIcon = (icone && icone.contains(e.target) && icone.style.display !== 'none');
        
        if (!isHeader && !isIcon) return;

        e.preventDefault();
        isDragging = true;
        isRealMovement = false; 
        
        startX = e.clientX;
        startY = e.clientY;
        currentX = 0;
        currentY = 0;
        
        const rect = element.getBoundingClientRect();
        initialLeft = rect.left;
        initialTop = rect.top;
        
        if (header) header.style.cursor = 'grabbing';
        if (icone) icone.style.cursor = 'grabbing';
        if (isIcon) element.style.cursor = 'grabbing';

        document.addEventListener('mousemove', onPointerMove, { passive: false });
        document.addEventListener('mouseup', onPointerUp);
    }

    function onPointerMove(e) {
        if (!isDragging) return;
        e.preventDefault();

        const dx = e.clientX - startX;
        const dy = e.clientY - startY;

        if (!isRealMovement) {
            if (Math.hypot(dx, dy) < 5) return;
            isRealMovement = true;
            
            element.style.bottom = 'auto';
            element.style.right = 'auto';
            element.style.left = initialLeft + 'px';
            element.style.top = initialTop + 'px';
            element.style.transition = 'none';
            element.style.willChange = 'transform';
            
            element.style.cursor = 'grabbing';
            if (header) header.style.cursor = 'grabbing';
            if (icone) icone.style.cursor = 'grabbing';
            
            overlay.style.display = 'block'; 
            rAF = requestAnimationFrame(updatePosition);
        }
        currentX = dx;
        currentY = dy;
    }

    function onPointerUp(e) {
        if (!isDragging) return;
        isDragging = false;

        if (rAF) cancelAnimationFrame(rAF);

        if (isRealMovement) {
            const finalLeft = initialLeft + currentX;
            const finalTop = initialTop + currentY;
            const maxLeft = window.innerWidth - element.offsetWidth;
            const maxTop = window.innerHeight - element.offsetHeight;
            const boundedLeft = Math.max(0, Math.min(finalLeft, maxLeft));
            const boundedTop = Math.max(0, Math.min(finalTop, maxTop));

            element.style.transform = '';
            element.style.willChange = 'auto';
            element.style.left = boundedLeft + 'px';
            element.style.top = boundedTop + 'px';
            
            overlay.style.display = 'none';
            salvarPosicaoPainelLocalStorage(storageKey, element);
            
            globalWasDragging = true;
            setTimeout(() => { globalWasDragging = false; }, 100);
        } else {
            globalWasDragging = false;
        }

        element.style.cursor = 'default';
        if (header) header.style.cursor = 'pointer';
        if (icone) icone.style.cursor = 'pointer';
        
        const conteudo = document.getElementById('pausa-script-conteudo');
        if (elementId === 'pausa-script-container' && conteudo && conteudo.style.display === 'none') {
             element.style.cursor = 'pointer';
        }

        document.removeEventListener('mousemove', onPointerMove);
        document.removeEventListener('mouseup', onPointerUp);
    }

    if (header) header.addEventListener('mousedown', onPointerDown);
    if (icone) icone.addEventListener('mousedown', onPointerDown);
}

function toggleVisibilidade() {
    const c=document.getElementById('pausa-script-container'),o=document.getElementById('pausa-script-conteudo'),i=document.getElementById('pausa-script-icone');
    if(!c||!o||!i)return;
    
    c.style.transform = 'none';

    const m=15;
    if (o.style.display !== 'none') {
        o.style.display = 'none'; i.style.display = 'flex';
        c.style.width = 'auto'; c.style.padding = '0'; c.style.cursor = 'pointer'; c.style.fontFamily = "'Segoe UI',Tahoma,Geneva,Verdana,sans-serif"; 
        c.style.background='transparent';
        c.style.border='none';
        c.style.boxShadow='none';
        c.style.backdropFilter='none';
        c.style.borderRadius='0';
    } else {
        o.style.display = 'block'; i.style.display = 'none';
        const lE = 300; c.style.width = lE + 'px'; c.style.padding = '15px'; c.style.cursor = 'default'; c.style.fontFamily = "'Segoe UI',Tahoma,Geneva,Verdana,sans-serif"; 
        c.style.background='linear-gradient(145deg, rgba(25, 30, 40, 0.9), rgba(35, 40, 55, 0.95))';
        c.style.border='1px solid rgba(0,255,255,0.3)';
        c.style.boxShadow='0 8px 20px rgba(0,0,0,0.5),0 0 10px rgba(0,255,255,0.1)';
        c.style.backdropFilter='blur(10px)';
        c.style.borderRadius='10px'; 
        const h = c.querySelector('#pausa-script-header'); if (h) h.style.cursor = 'pointer';
        if(c.style.top&&c.style.left){let cT=parseInt(c.style.top,10),cL=parseInt(c.style.left,10);setTimeout(()=>{const hE=c.offsetHeight;const mL=window.innerWidth-lE-m;if(cL>mL){c.style.left=mL+'px'}if(cL<m){c.style.left=m+'px'}const mT=window.innerHeight-hE-m;if(cT>mT){c.style.top=mT+'px'}if(cT<m){c.style.top=m+'px'}},10)}
        else { const r=c.getBoundingClientRect();c.style.left=r.left+'px';c.style.top=r.top+'px';c.style.bottom = ''; c.style.right = ''; }
    }
}

function toggleVisibilidadeConfig(){
    const c=document.getElementById('pausa-script-config-container');
    if(!c)return;
    isConfigPainelAberto=!isConfigPainelAberto;
    c.style.display=isConfigPainelAberto?'block':'none';
    
    // CORREÇÃO: Reseta a edição se fechar o painel
    if(!isConfigPainelAberto) {
         indiceEdicao = -1;
         const btn=document.getElementById('btn-adicionar-horario');
         const iH=document.getElementById('novo-horario-hora'),iS=document.getElementById('novo-horario-status');
         if(btn) { btn.innerHTML = "+"; btn.style.background = ""; btn.style.color = ""; }
         if(iH) iH.value=''; if(iS) iS.value='';
    } else {
         renderizarAgendamentos();renderizarHistoricoLog();renderizarEstatisticas();
    }
}

function nomeExibicaoStatus(status) {
    const n = normalizarStatusTexto(status);
    if (n === 'pausa auricular' || n === 'pausa pessoal' || n === 'descanso') return 'Descanso';
    return status;
}

function statusParaAcao(status) {
    const n = normalizarStatusTexto(status);
    if (n === 'descanso') return 'Pausa auricular';
    return status;
}

function renderizarAgendamentos(){const l=document.getElementById('agendamento-lista');if(!l)return; l.innerHTML='';if(PAUSAS_AGENDADAS.length===0){l.innerHTML='<p style="color:#999;margin:5px 0;font-size:11px;text-align:center;">Nenhum horário agendado.</p>'}else{const r=document.createDocumentFragment();PAUSAS_AGENDADAS.forEach((p,idx)=>{const i=document.createElement('div');i.style.cssText=`display:flex;justify-content:space-between;align-items:center;border-bottom:1px dashed #ffffff15;padding:4px 2px;font-size:11px;`;const tD=document.createElement('div');tD.style.flexGrow='1';tD.innerHTML=`<strong>${p.hora}</strong>: ${nomeExibicaoStatus(p.status)}`;const bD=document.createElement('div');bD.style.flexShrink='0';bD.style.marginLeft='10px'; const bE=document.createElement('button');bE.innerHTML='✏️';bE.title=`Editar ${p.hora}`;bE.style.cssText=`background:0 0;border:none;color:#ffc107;cursor:pointer;font-size:14px;padding:0 4px;opacity:.7;transition:opacity .2s;`;bE.onmouseover=function(){this.style.opacity='1'};bE.onmouseout=function(){this.style.opacity='.7'};bE.onclick=()=>carregarHorarioParaEdicao(idx); const bR=document.createElement('button');bR.innerHTML='❌';bR.title=`Remover ${p.hora}`;bR.style.cssText=`background:0 0;border:none;color:#ff6b6b;cursor:pointer;font-size:14px;padding:0 4px;margin-left:5px;opacity:.7;transition:opacity .2s;`;bR.onmouseover=function(){this.style.opacity='1'};bE.onmouseout=function(){this.style.opacity='.7'};bR.onclick=()=>removerHorario(idx); bD.appendChild(bE);bD.appendChild(bR);i.appendChild(tD);i.appendChild(bD);r.appendChild(i)});l.appendChild(r)}}
function iconeStatusPausa(status) {
    const n = normalizarStatusTexto(status);
    if (n === 'na fila') return '🎧';
    if (n === 'refeicao') return '🍽️';
    if (n === 'descanso' || n === 'pausa auricular' || n === 'pausa pessoal') return '☕';
    if (n === 'pausa out') return '🛑';
    if (n === 'treinamento') return '📚';
    if (n === 'backoffice' || n === 'atividade backoffice' || n === 'ativ. backoffice') return '📁';
    if (n === 'feedback') return '📝';
    if (n === 'particular') return '🚽';
    if (n === 'saude' || n === 'questoes de saude') return '🩺';
    if (n === 'reuniao') return '👥';
    if (n === 'disponivel') return '🟢';
    if (n === 'logoff') return '🏠';
    return '•';
}

function renderizarHistoricoLog(){
    const l=document.getElementById('historico-log');
    if(!l)return;
    l.innerHTML='';
    if(historicoPausas.length===0){
        l.innerHTML='<p style="color:#999;margin:5px 0;font-size:11px;text-align:center;">Nenhum registro ainda.</p>';
        return;
    }

    const itens = historicoPausas.slice().reverse();
    const f=document.createDocumentFragment();

    itens.forEach((p,idx)=>{
        const nome = nomeExibicaoStatus(p.tipo);
        const card=document.createElement('div');
        card.style.cssText=`display:grid;grid-template-columns:24px 1fr 96px;gap:8px;align-items:center;padding:7px 6px;border-radius:8px;margin-bottom:6px;background:rgba(255,255,255,.035);border:1px solid rgba(255,255,255,.06);font-size:11px;line-height:1.25;`;
        if(idx===0){
            card.style.background='rgba(0,191,255,.08)';
            card.style.borderColor='rgba(0,191,255,.18)';
        }
        card.innerHTML=`
            <div style="width:24px;height:24px;border-radius:8px;display:flex;align-items:center;justify-content:center;background:rgba(0,0,0,.28);font-size:13px;">${iconeStatusPausa(nome)}</div>
            <div style="min-width:0;">
                <div style="display:flex;align-items:center;gap:6px;min-width:0;">
                    <strong style="color:#f4f8fb;white-space:nowrap;overflow:hidden;text-overflow:ellipsis;">${nome}</strong>
                    ${idx===0 ? '<span style="font-size:9px;color:#00d4ff;border:1px solid rgba(0,212,255,.3);border-radius:999px;padding:1px 5px;background:rgba(0,212,255,.08);">última</span>' : ''}
                </div>
                <div style="color:#a9b7c6;margin-top:2px;">${p.inicio} → ${p.fim}</div>
            </div>
            <div style="min-width:96px;text-align:right;color:#36c6ff;font-weight:800;font-size:13px;line-height:1.1;font-family:'Consolas','Courier New',monospace;font-variant-numeric:tabular-nums;letter-spacing:.35px;white-space:nowrap;padding:3px 0 3px 8px;">${p.duracao}</div>
        `;
        f.appendChild(card);
    });
    l.appendChild(f);
}

function renderizarEstatisticas(){
    const s=document.getElementById('estatisticas-pausas');
    if(!s)return;
    s.innerHTML='';
    if(historicoPausas.length===0){
        s.innerHTML='<div style="background:rgba(0,0,0,.25);border:1px solid rgba(0,255,255,.12);border-radius:8px;padding:10px;color:#999;font-style:italic;font-size:11px;text-align:center;">Sem estatísticas.</div>';
        return;
    }

    const r={};
    historicoPausas.forEach(p=>{
        const t=nomeExibicaoStatus(p.tipo);
        if(!r[t])r[t]={c:0,s:0};
        r[t].c++;
        r[t].s+=converterDuracaoParaSegundos(p.duracao);
    });

    const d=document.createElement('details');
    d.style.marginTop='10px';
    d.open=!1;
    const m=document.createElement('summary');
    m.textContent='Estatísticas';
    d.appendChild(m);

    const c=document.createElement('div');
    c.style.cssText=`background-color:rgba(0,0,0,.25);padding:8px;border-radius:0 0 8px 8px;font-size:11px;`;
    const f=document.createDocumentFragment();

    const k=Object.keys(r).sort((a,b)=>r[b].s-r[a].s || a.localeCompare(b));
    k.forEach((t,x)=>{
        const o=r[t];
        const u=formatarDuracao(o.s);
        const i=document.createElement('div');
        i.style.cssText=`display:grid;grid-template-columns:24px 1fr 96px;gap:8px;align-items:center;padding:6px 4px;border-bottom:1px dashed rgba(255,255,255,.10);line-height:1.25;`;
        if(x===k.length-1)i.style.borderBottom='none';
        i.innerHTML=`
            <span style="width:22px;height:22px;border-radius:7px;display:flex;align-items:center;justify-content:center;background:rgba(0,0,0,.28);font-size:13px;">${iconeStatusPausa(t)}</span>
            <strong style="min-width:0;overflow:hidden;text-overflow:ellipsis;white-space:nowrap;color:#f4f8fb;">${t} <span style="color:#93a4b6;font-weight:600;">(x${o.c})</span></strong>
            <span style="min-width:96px;text-align:right;color:#36c6ff;font-weight:800;font-size:13px;line-height:1.1;font-family:'Consolas','Courier New',monospace;font-variant-numeric:tabular-nums;letter-spacing:.35px;white-space:nowrap;padding-left:8px;">${u}</span>
        `;
        f.appendChild(i);
    });
    c.appendChild(f);
    d.appendChild(c);
    s.appendChild(d);
}
function verificarSeletoresEAtualizarUI(){const rD=document.getElementById('seletores-resultados'); if(!rD)return; rD.innerHTML='<p style="font-style:italic;color:#bbb;">Verificando...</p>'; let hR='<ul style="list-style:none;padding:0;margin:5px 0 0 0;font-size:11px;">';setTimeout(()=>{Object.keys(SELECTORES).forEach(k=>{const s=SELECTORES[k]; const e=document.querySelector(s);const f=e!==null&&(e.offsetParent!==null||e.tagName==='DIV'||e.tagName==='SPAN'); hR+=`<li style="padding:3px 0;border-bottom:1px dashed #ffffff15;display:flex;justify-content:space-between;align-items:center;"><span style="color:#ccc; max-width: 80px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap;">${k}:</span><span><code style="font-size:10px;background-color:rgba(0,255,255,.1);border:1px solid rgba(0,255,255,.2);color:#9fefef;padding:1px 3px;border-radius:3px; max-width: 120px; display: inline-block; overflow: hidden; text-overflow: ellipsis; white-space: nowrap;">${s}</code><span style="margin-left:8px;font-weight:700;color:${f?'#50fa7b':'#ff5555'};">${f?'Ok':'Fail'}</span></span></li>`;});hR+='</ul>';rD.innerHTML=hR},50)}

// --- CRIAR UI PRINCIPAL (VISUAL NÍTIDO) ---
function criarUI() {
    const cId='pausa-script-container';let cA=document.getElementById(cId);if(cA)cA.remove(); const c=document.createElement('div');c.id=cId;document.body.appendChild(c);
    const pS=carregarPosicaoPainelLocalStorage(STORAGE_KEY_POS_MAIN);
    let cssPos = `position:fixed; ${pS ? (pS.top ? `top:${pS.top};left:${pS.left};` : `bottom:${pS.bottom};right:${pS.right};`) : 'bottom:30px;right:30px;'}`;
    
    c.style.cssText = `${cssPos} width:auto;height:auto;color:#E0E0E0;font-family:'Segoe UI',Tahoma,Geneva,Verdana,sans-serif;padding:0;border-radius:0;border:none;background:transparent;box-shadow:none;backdrop-filter:none;z-index:2147483647;cursor:pointer;user-select:none;transition:box-shadow .2s;box-sizing:border-box;will-change:transform;`;
    
    c.innerHTML=`<style>
        #${cId} *{box-sizing:border-box}
        #${cId} .script-btn{display:flex;align-items:center;justify-content:center;gap:10px;padding:10px 15px;border:none;border-radius:8px;cursor:pointer;font-size:13px;font-weight:600;color:#FFFFFF;text-shadow:0 1px 2px rgba(0,0,0,0.3);background-image:none;box-shadow:0 3px 5px rgba(0,0,0,0.3),inset 0 1px 1px rgba(255,255,255,0.1);transition:all 0.15s ease-out;width:100%}
        #${cId} .script-btn:hover{filter:brightness(1.1);transform:translateY(-2px)}
        #${cId} .script-btn:active{transform:translateY(1px);filter:brightness(0.95)}
        #${cId} #btn-disponivel,#${cId} #btn-toggle-agendamento[style*="rgb(35, 167, 69)"]{background:linear-gradient(135deg,#23a745,#2dc24f)}
        #${cId} #btn-toggle-agendamento[style*="rgb(220, 53, 69)"]{background:linear-gradient(135deg,#dc3545,#e84a5f)}
        #${cId} #btn-pause-resume-agendamento[style*="rgb(255, 193, 7)"]{background:linear-gradient(135deg,#ffc107,#ffd25a);color:#fff;text-shadow:0 1px 2px rgba(0,0,0,0.45)}
        #${cId} #btn-pause-resume-agendamento[style*="rgb(0, 127, 255)"]{background:linear-gradient(135deg,#007FFF,#009cff);color:#fff}
        #${cId} #btn-na-fila{background:linear-gradient(135deg,#0096C7,#00bfff)}
        #${cId} #btn-refeicao{background:linear-gradient(135deg,#fd7e14,#ff9a40)}
        #${cId} #btn-descanso{background:linear-gradient(135deg,#ffc107,#ffd25a);color:#fff;text-shadow:0 1px 2px rgba(0,0,0,0.45)}
        #${cId} #btn-pausa-out{background:linear-gradient(135deg,#e63946,#f45b69)}
        #${cId} #btn-treinamento{background:linear-gradient(135deg,#6a00c9,#8e2de2)}
        #${cId} #btn-backoffice{background:linear-gradient(135deg,#c94b4b,#d46a6a)}
        #${cId} #btn-feedback{background:linear-gradient(135deg,#e85d04,#fa7e30)}
        #${cId} #btn-particular{background:linear-gradient(135deg,#8338ec,#9f60f0)}
        #${cId} #btn-saude{background:linear-gradient(135deg,#5e60ce,#7b7de0)}
        #${cId} #btn-reuniao{background:linear-gradient(135deg,#00c9c9,#30dede);color:#fff;text-shadow:0 1px 2px rgba(0,0,0,0.45)}
        #${cId} #btn-logoff{background:linear-gradient(135deg,#6c757d,#868e96)}
        #${cId} #pausa-script-header{display:flex;justify-content:space-between;align-items:center;margin-bottom:12px;padding-bottom:8px;border-bottom:1px solid rgba(0,255,255,0.2);cursor:pointer}
        #${cId} #pausa-script-header h3{color:#0FF;margin:0;font-size:16px;font-weight:600}
        #${cId} #pausa-script-header span#btn-abrir-config{cursor:pointer;color:#0FF;font-size:20px;margin-right:12px}
        #${cId} #pausa-script-header span#btn-minimizar{cursor:pointer;color:#e63946;font-size:16px;font-weight:700}
        #${cId} #status-display{background-color:rgba(0,0,0,0.2);border:1px solid rgba(0,0,0,0.3);color:#FFF;padding:10px;border-radius:8px;margin-bottom:15px;text-align:center;font-size:14px}
        #${cId} #current-status{font-weight:700}
        #${cId} #pausa-script-icone{display:flex;align-items:center;justify-content:center;width:auto;height:auto;cursor:pointer;background:transparent;border:none;box-shadow:none;padding:0;margin:0;}
        #${cId} #top-bar-time{font-family:'Segoe UI',Tahoma,Geneva,Verdana,sans-serif;font-size:25px;font-weight:700;letter-spacing:1px;color:#00d9ff;background:transparent;border:none;box-shadow:none;padding:0;margin:0;line-height:1;white-space:nowrap;font-variant-numeric:tabular-nums;}
        #${cId} #pausa-script-clock-dot{display:none!important;width:0;height:0;opacity:0;pointer-events:none;}
        @keyframes blink-animation{0%{opacity:1}50%{opacity:0.3}100%{opacity:1}}
        </style>
    
        <div id="pausa-script-icone"><span id="top-bar-time">--:--:--</span></div>
        <div id="pausa-script-conteudo" style="display:none;padding:0;">
            <div id="pausa-script-header">
                <h3>🛠️ Gerenciador</h3>
                <div style="display:flex;align-items:center;gap:12px;transform:translateY(3px);">
                    <span id="pausa-script-header-clock" style="font-size:16px;font-family:'Courier New',Courier,monospace;color:#f1fa8c;line-height:1;">--:--:--</span>
                    <div style="flex-shrink:0;"><span id="btn-abrir-config" title="Configurações">⚙️</span><span id="btn-minimizar" title="Fechar Painel">❌</span></div>
                </div>
            </div>
            <div id="status-display">Status: <strong id="current-status" style="color:#50fa7b;">${estadoAtual.tipo}</strong></div>
            <div style="margin-bottom:15px;">
                <button id="btn-toggle-agendamento" class="script-btn">Ligar Automação</button>
                <button id="btn-pause-resume-agendamento" class="script-btn" style="display:none; margin-top: 8px;">Pausar ⏸️</button>
            </div>
            <div id="buttons-container" style="display:grid;grid-template-columns:repeat(2,1fr);gap:10px;margin-bottom:10px;">
                <button id="btn-disponivel" class="script-btn">🟢 Disponível</button>
                <button id="btn-na-fila" class="script-btn">🎧 Na Fila</button>
                <button id="btn-refeicao" class="script-btn">🍽️ Refeição</button>
                <button id="btn-descanso" class="script-btn">☕ Descanso</button>
                <button id="btn-pausa-out" class="script-btn">🛑 Pausa Out</button>
                <button id="btn-treinamento" class="script-btn">📚 Treinamento</button>
                <button id="btn-backoffice" class="script-btn">📁 Backoffice</button>
                <button id="btn-feedback" class="script-btn">📝 Feedback</button>
                <button id="btn-particular" class="script-btn">🚽 Particular</button>
                <button id="btn-saude" class="script-btn">🩺 Saúde</button>
            </div>
            <div id="buttons-extra" style="display:grid;grid-template-columns:1fr 1fr;gap:10px;">
                <button id="btn-reuniao" class="script-btn">🧑‍🤝‍🧑 Reunião</button>
                <button id="btn-logoff" class="script-btn">🏠 Fazer Logoff</button>
            </div>
        </div>`;
    
    criarDragOverlay(); 
    setupGPUDrag('pausa-script-container', 'pausa-script-header', STORAGE_KEY_POS_MAIN);

    c.addEventListener('click',function(e){
        if (globalWasDragging) return;
        
        const t=e.target.closest('button'),n=e.target.closest('span#btn-minimizar, span#btn-abrir-config'),o=e.target.closest('div#pausa-script-icone');let l=t||n||o;if(!l)return; 
        const d=document.getElementById('pausa-script-conteudo');
        if (o && d && d.style.display==='none'){ toggleVisibilidade(); return; }
        const a=document.getElementById('pausa-script-header');
        if (d.style.display!=='none'&&a&&a.contains(e.target)&&!n&&!t)return;
        
        const i=l.id;switch(i){
            case'btn-na-fila':iniciarNaFila(false);break; 
            case'btn-refeicao':iniciarRefeicao(false);break;
            case'btn-descanso':iniciarDescanso(false);break; 
            case'btn-pausa-out':iniciarPausaOut(false);break;
            case'btn-disponivel':voltarDisponivel(false);break; 
            case'btn-logoff':realizarLogoff(false);break;
            case'btn-treinamento':iniciarTreinamento(false);break; 
            case'btn-reuniao':iniciarReuniao(false);break;
            case'btn-backoffice':iniciarBackoffice(false);break;
            case'btn-feedback':iniciarFeedback(false);break;
            case'btn-particular':iniciarParticular(false);break;
            case'btn-saude':iniciarQuestoesSaude(false);break;
            case'btn-toggle-agendamento':toggleAgendamento();break; 
            case'btn-pause-resume-agendamento':togglePausaAutomacao();break;
            case'btn-minimizar':toggleVisibilidade();break; 
            case'btn-abrir-config':toggleVisibilidadeConfig();break;
        }
    });
    
    const bTg=c.querySelector('#btn-toggle-agendamento'),bP=c.querySelector('#btn-pause-resume-agendamento');
    if(bTg&&bP){
        if(isAgendamentoAtivo){
            bTg.textContent="Desligar Automação";bTg.style.background = 'linear-gradient(135deg, #dc3545, #e84a5f)';bP.style.display='block';
            if(isAutomacaoPausada){bP.textContent="Retomar ▶️";bP.style.background = 'linear-gradient(135deg, #007FFF, #009cff)';bP.style.color = '#fff';}
            else{bP.textContent="Pausar ⏸️";bP.style.background = 'linear-gradient(135deg, #ffc107, #ffd25a)';bP.style.color = '#fff';bP.style.textShadow = '0 1px 2px rgba(0,0,0,0.45)';}
        }else{
            bTg.textContent="Ligar Automação";bTg.style.background = 'linear-gradient(135deg, #23a745, #2dc24f)';bP.style.display='none';
        }
    }
    
    atualizarUI();
    adicionarAtalho(); 
    setInterval(atualizarUI,1000);
}

// --- CRIAR UI CONFIGURAÇÕES ---
function criarUIConfiguracoes() {
    const cId='pausa-script-config-container';let cA=document.getElementById(cId);if(cA)cA.remove(); const cC=document.createElement('div');cC.id=cId;document.body.appendChild(cC); const pSC=carregarPosicaoPainelLocalStorage(STORAGE_KEY_POS_CONFIG);
    let cssPos = `position:fixed; ${pSC ? (pSC.top ? `top:${pSC.top};left:${pSC.left};` : `bottom:${pSC.bottom};right:${pSC.right};`) : 'bottom:30px;right:340px;'}`;
    
    cC.style.cssText=`${cssPos} width:340px;background:linear-gradient(145deg,rgba(25,30,40,0.9),rgba(35,40,55,0.95));color:#E0E0E0;border:1px solid rgba(0,255,255,.3);border-radius:10px;padding:15px;z-index:2147483646;box-shadow:0 8px 20px rgba(0,0,0,0.5),0 0 10px rgba(0,255,255,0.1);font-family:'Segoe UI',Tahoma,Geneva,Verdana,sans-serif;font-size:13px;height:auto;max-height:calc(100vh - 60px);backdrop-filter:blur(10px);transition:box-shadow .2s;display:none;overflow-y:auto;overflow-x:hidden;box-sizing:border-box;cursor:default;will-change:transform;color-scheme:dark;scrollbar-gutter:stable;`;
    
    cC.innerHTML=`<style>
        #${cId}, #${cId} *{box-sizing:border-box;scrollbar-width:thin;scrollbar-color:rgba(0,191,255,.55) rgba(0,0,0,.26)}
        #${cId}::-webkit-scrollbar, #${cId} *::-webkit-scrollbar{width:10px;height:10px}
        #${cId}::-webkit-scrollbar-track, #${cId} *::-webkit-scrollbar-track{background:rgba(0,0,0,.28);border-radius:999px}
        #${cId}::-webkit-scrollbar-thumb, #${cId} *::-webkit-scrollbar-thumb{background:linear-gradient(180deg,rgba(0,191,255,.55),rgba(0,140,220,.42));border-radius:999px;border:2px solid rgba(0,0,0,.18)}
        #${cId}::-webkit-scrollbar-thumb:hover, #${cId} *::-webkit-scrollbar-thumb:hover{background:linear-gradient(180deg,rgba(0,210,255,.75),rgba(0,160,235,.58))}
        #${cId}::-webkit-scrollbar-corner, #${cId} *::-webkit-scrollbar-corner{background:rgba(0,0,0,.22)}
        #${cId} input, #${cId} select, #${cId} textarea{color-scheme:dark}
        #${cId} .script-btn-config{padding:10px 15px;border:none;border-radius:8px;cursor:pointer;font-size:13px;font-weight:600;color:#fff;text-shadow:0 1px 2px rgba(0,0,0,0.3);box-shadow:0 3px 5px rgba(0,0,0,0.3),inset 0 1px 1px rgba(255,255,255,0.1);transition:all .15s ease-out;width:100%}
        #${cId} .script-btn-config:hover{filter:brightness(1.1);transform:translateY(-2px)}
        #${cId} details{margin-top:15px;border:1px solid rgba(0,255,255,.1);border-radius:8px;background-color:rgba(0,0,0,.2);overflow:hidden}
        #${cId} summary{padding:10px 12px;cursor:pointer;color:#0FF;font-weight:600;list-style:none;display:flex;align-items:center}
        #${cId} summary::before{content:'▶';margin-right:8px;font-size:10px;transition:transform .2s;display:inline-block}
        #${cId} details[open]>summary::before{transform:rotate(90deg)}
        #${cId} details>div{padding:12px;border-top:1px solid rgba(0,255,255,.1)}
        #${cId} .add-schedule-form{display:flex;gap:8px;margin-top:10px;align-items:center;min-width:0}
        #${cId} .add-schedule-form input{background-color:rgba(0,0,0,.32);border:1px solid rgba(0,255,255,.22);color:#E0E0E0;padding:8px;border-radius:6px;font-size:13px;min-width:0;outline:none;box-shadow:inset 0 1px 2px rgba(0,0,0,.35)}
        #${cId} .add-schedule-form input:focus, #${cId} #notif-tempo:focus{border-color:rgba(0,255,255,.5);box-shadow:0 0 0 2px rgba(0,191,255,.12), inset 0 1px 2px rgba(0,0,0,.35)}
        #${cId} #novo-horario-hora{width:98px;flex:0 0 98px}
        #${cId} #novo-horario-status{flex:1 1 auto;min-width:0}
        #${cId} .add-schedule-form button{flex-shrink:0;padding:8px 14px!important;width:auto!important;background:linear-gradient(135deg,#23a745,#2dc24f)!important;font-weight:700;border:none;border-radius:8px;transition:all 0.3s;color:#fff;text-shadow:0 1px 2px rgba(0,0,0,.4)}
        #${cId} #agendamento-lista, #${cId} #historico-log{max-width:100%;overflow-y:auto;overflow-x:hidden;margin-bottom:10px;border-radius:8px;padding:8px;background-color:rgba(0,0,0,.25);box-shadow:inset 0 1px 3px rgba(0,0,0,.25)}
        #${cId} #historico-log{margin-bottom:0}
        #${cId} #agendamento-lista button{background:0 0;border:none;cursor:pointer;font-size:14px;padding:0 4px;margin-left:5px}
        #${cId} #notif-tempo{width:58px;background-color:rgba(0,0,0,.32);border:1px solid rgba(0,255,255,.22);color:#E0E0E0;padding:6px 8px;border-radius:6px}
        #${cId} input[type='checkbox']{accent-color:#00bfff}
        #${cId} #estatisticas-pausas details{margin-top:0}
        #${cId} #estatisticas-pausas summary{border-bottom:1px solid rgba(0,255,255,.1)}
        </style>
        
        <div id="pausa-script-config-header" style="display:flex;justify-content:space-between;align-items:center;margin-bottom:10px;padding-bottom:5px;border-bottom:1px solid rgba(0,255,255,.2);cursor:pointer;"><h3 style="margin:0;font-size:15px;">⚙️ Config & Histórico</h3><span id="btn-fechar-config" style="cursor:pointer;color:#e63946;font-weight:700;">❌</span></div>
        <div id="estatisticas-pausas" style="margin-bottom:0;"></div>
        
        <details><summary>🔔 Configurar Notificações</summary><div id="notificacao-config"><div style="display:flex;justify-content:space-between;margin-bottom:8px;"><label for="notif-ativadas">Ativar Visuais:</label><input type="checkbox" id="notif-ativadas"></div><div style="display:flex;justify-content:space-between;margin-bottom:8px;"><label for="notif-som">Ativar Som:</label><input type="checkbox" id="notif-som"></div><div style="display:flex;justify-content:space-between;"><label for="notif-tempo">Antecedência (s):</label><input type="number" id="notif-tempo" min="5" max="55" step="5" style="width:50px;"></div></div></details>
        <details open><summary>Editar Horários</summary><div id="agendamento-edit"><div id="agendamento-lista"></div><div class="add-schedule-form"><input type="time" id="novo-horario-hora" required><input type="text" id="novo-horario-status" placeholder="Ex: Refeição" required style="flex-grow:1;"><button id="btn-adicionar-horario">+</button></div></div></details>
        <details><summary>Histórico de Pausas</summary><div id="historico-log" style="max-height:190px;overflow-y:auto;"></div></details>
    `;
    
    setupGPUDrag('pausa-script-config-container', 'pausa-script-config-header', STORAGE_KEY_POS_CONFIG);

    const bF=cC.querySelector('#btn-fechar-config');if(bF)bF.onclick=toggleVisibilidadeConfig;
    const bA=cC.querySelector('#btn-adicionar-horario');if(bA)bA.onclick=adicionarNovoHorario;
    const cN=cC.querySelector('#notif-ativadas'),cS=cC.querySelector('#notif-som'),iT=cC.querySelector('#notif-tempo');if(cN)cN.checked=configNotificacao.ativadas;if(cS)cS.checked=configNotificacao.somAtivado;if(iT)iT.value=configNotificacao.antecedenciaSegundos;
    function sM(){configNotificacao.ativadas=cN.checked;configNotificacao.somAtivado=cS.checked;let t=parseInt(iT.value,10);if(isNaN(t)||t<5)t=5;if(t>55)t=55;iT.value=t;configNotificacao.antecedenciaSegundos=t;salvarConfiguracoesNotificacaoLocalStorage(configNotificacao);}
    if(cN)cN.onchange=sM;if(cS)cS.onchange=sM;if(iT)iT.onchange=sM;

    renderizarAgendamentos();renderizarHistoricoLog();renderizarEstatisticas();
}


// --- LÓGICA DE CORES NÍTIDAS E AJUSTADAS (V56.12) ---
function atualizarUI() {
    const a=new Date,h=a.toLocaleTimeString('pt-BR',{hour:'2-digit',minute:'2-digit',second:'2-digit'});
    
    const tFlutuante=document.getElementById('top-bar-time');
    const tDot=document.getElementById('pausa-script-clock-dot');
    const tHeader = document.getElementById('pausa-script-header-clock');
    const sDisplay = document.getElementById('current-status');

    const statusLower = estadoAtual.tipo.toLowerCase();
    
    // --- DEFINIÇÃO DA COR NÍTIDA ---
    let corFinal = '#ffb347'; 

    if (statusLower === 'disponível') {
        corFinal = '#50fa7b'; 
    } else if (statusLower === 'na fila') {
        corFinal = '#40e0ff'; 
    } else if (statusLower.includes('pausa out') || statusLower.includes('backoffice') || statusLower.includes('feedback') || statusLower.includes('limpeza') || statusLower.includes('reunião')) {
        corFinal = '#e63946'; 
    } else if (statusLower === 'logoff') {
        corFinal = '#999';
    }

    // APLICAÇÃO VISUAL
    if(tFlutuante){
        tFlutuante.textContent = h;
        tFlutuante.style.color = corFinal;
        tFlutuante.style.textShadow = 'none';
    }
    
    if(tHeader) {
        tHeader.textContent = h;
        tHeader.style.color = corFinal;
        tHeader.style.textShadow = 'none'; 
    }

    if (tDot) {
        tDot.style.display = 'none';
        tDot.style.animation = 'none';
        tDot.style.opacity = '0';
    }

    if(sDisplay){
        let e=''; if(estadoAtual.ativa){const o=a.getTime()-estadoAtual.inicio.getTime();e=` (${formatarDuracao(Math.max(0,Math.round(o/1000)))})`;}
        sDisplay.textContent=`${estadoAtual.tipo}${e}`;
        sDisplay.style.color = corFinal;
    }
}

// ----------------------------------------------------
// START
// ----------------------------------------------------
console.log("[GERENCIADOR FUSÃO] Iniciando (V56.19: Confirmação silenciosa + retry)...");
setTimeout(() => {
    try {
        configNotificacao = carregarConfiguracoesNotificacaoLocalStorage();
        PAUSAS_AGENDADAS = carregarHorariosLocalStorage().map(p => ({
                ...p,
                status: normalizarStatusTexto(p.status) === 'pausa auricular' ? 'Descanso' : p.status
            }));
            salvarHorariosLocalStorage(PAUSAS_AGENDADAS);
        
        // --- RECUPERAÇÃO DE ESTADO ---
        recuperarEstadoInicial();
        
        if (configNotificacao.ativadas && "Notification" in window && Notification.permission === 'default') Notification.requestPermission();
        
        criarUI();
        criarUIConfiguracoes();
        renderizarAgendamentos();
        
        audioNotificacao.volume = 0.5;
        document.addEventListener('click', desbloquearAudio, { once: true });
    } catch (error) { console.error("[GERENCIADOR] Erro:", error); }
}, 2800);

})();

