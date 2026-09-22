# python-renan_servidor_gdt.py
renan_servidor_gdt
nano renan_servidor_gdt.py

from http.server import HTTPServer, BaseHTTPRequestHandler
import xml.etree.ElementTree as ET
import statistics

# ============================================================
# EXEMPLO DE ARQUIVO GDT.XML
# ============================================================
SAMPLE_GDT = """<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE gretldata SYSTEM "gretldata.dtd">
<gretldata version="1.3" name="dados_exemplo" frequency="1" startobs="1" endobs="10" type="cross-section">
<description>Dados de exemplo - Renda, Educacao, Idade e Genero</description>
<variables count="5">
<variable name="const" label="Constante" />
<variable name="renda" label="Renda mensal (R$)" />
<variable name="educ" label="Anos de estudo" discrete="true" />
<variable name="idade" label="Idade em anos" discrete="true" />
<variable name="genero" label="Genero (1=F, 0=M)" discrete="true" />
</variables>
<observations count="10" labels="false">
<obs>1 3500 12 28 1</obs>
<obs>1 4200 16 32 0</obs>
<obs>1 2800 11 25 1</obs>
<obs>1 5500 18 35 0</obs>
<obs>1 3200 14 30 1</obs>
<obs>1 4800 17 33 0</obs>
<obs>1 2500 10 22 1</obs>
<obs>1 6200 20 40 0</obs>
<obs>1 3800 15 29 1</obs>
<obs>1 4500 16 34 0</obs>
</observations>
</gretldata>"""

# ============================================================
# ANÁLISE DO GDT
# ============================================================
def analisar_gdt(xml_content):
    resultado = {
        'erro': None, 'eh_gretl': False, 'metadados': {},
        'descricao': '', 'variaveis': [], 'observacoes': [],
        'estatisticas': {}, 'total_elementos': 0
    }
    try:
        root = ET.fromstring(xml_content)
    except ET.ParseError as e:
        resultado['erro'] = str(e)
        return resultado
    
    resultado['total_elementos'] = len(list(root.iter()))
    
    if root.tag == 'gretldata':
        resultado['eh_gretl'] = True
        for attr in ['version', 'name', 'frequency', 'startobs', 'endobs', 'type']:
            if attr in root.attrib:
                resultado['metadados'][attr] = root.attrib[attr]
        
        desc = root.find('description')
        if desc is not None and desc.text:
            resultado['descricao'] = desc.text.strip()
        
        vars_elem = root.find('variables')
        if vars_elem is not None:
            resultado['metadados']['qtd_variaveis'] = vars_elem.get('count', '?')
            for var in vars_elem.findall('variable'):
                resultado['variaveis'].append({
                    'nome': var.get('name', '?'),
                    'label': var.get('label', ''),
                    'discreta': var.get('discrete', 'false') == 'true'
                })
        
        obs_elem = root.find('observations')
        if obs_elem is not None:
            resultado['metadados']['qtd_observacoes'] = obs_elem.get('count', '?')
            for obs in obs_elem.findall('obs'):
                if obs.text:
                    resultado['observacoes'].append(obs.text.strip().split())
        
        if resultado['variaveis'] and resultado['observacoes']:
            for i, var in enumerate(resultado['variaveis']):
                if var['nome'] == 'const':
                    continue
                try:
                    valores = [float(o[i]) for o in resultado['observacoes'] if len(o) > i and o[i].strip()]
                    if valores:
                        resultado['estatisticas'][var['nome']] = {
                            'media': round(statistics.mean(valores), 2),
                            'mediana': round(statistics.median(valores), 2),
                            'min': round(min(valores), 2),
                            'max': round(max(valores), 2),
                            'desvio': round(statistics.stdev(valores), 2) if len(valores) > 1 else 0,
                            'n': len(valores)
                        }
                except:
                    pass
    else:
        resultado['metadados']['tag_raiz'] = root.tag
        for elem in list(root)[:15]:
            resultado['variaveis'].append({
                'nome': elem.tag, 'label': f'Atributos: {len(elem.attrib)}', 'discreta': False
            })
            if elem.text and elem.text.strip():
                resultado['observacoes'].append([elem.text.strip()[:50]])
    
    return resultado

def formatar_xml(xml_content):
    try:
        root = ET.fromstring(xml_content)
        ET.indent(root, space='  ')
        return ET.tostring(root, encoding='unicode', xml_declaration=True)
    except:
        return xml_content

def escape_html(text):
    return text.replace('&', '&amp;').replace('<', '&lt;').replace('>', '&gt;')

# ============================================================
# HTML BASE
# ============================================================
HTML_BASE = """<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>GDT XML Analisador</title>
<style>
* { box-sizing: border-box; margin: 0; padding: 0; }
body {
    font-family: system-ui, -apple-system, sans-serif;
    background: #1a1a2e; color: #e8e5df; line-height: 1.6;
    padding: 24px 16px;
}
.container { max-width: 960px; margin: 0 auto; }
.header { text-align: center; margin-bottom: 28px; }
.header h1 { font-size: 28px; margin-bottom: 6px; }
.gradient { background: linear-gradient(100deg, #e94560, #53a8b6); 
    -webkit-background-clip: text; background-clip: text; color: transparent; }
.sub { color: #9a978f; font-size: 14px; }
.card {
    background: #16213e; border: 1px solid rgba(232,229,223,.12);
    border-radius: 14px; padding: 20px; margin-bottom: 20px;
}
.card h2 { font-size: 17px; margin-bottom: 16px; display: flex; align-items: center; gap: 10px; }
.card h3 { font-size: 14px; margin: 20px 0 10px; color: #53a8b6; }
.upload-area { display: flex; flex-wrap: wrap; gap: 10px; align-items: center; }
input[type=file] { display: none; }
.file-label {
    display: inline-flex; align-items: center; gap: 8px;
    background: #0f3460; color: #e8e5df; border: 2px dashed rgba(232,229,223,.15);
    padding: 14px 22px; border-radius: 10px; cursor: pointer; font-weight: 500;
    transition: all .2s;
}
.file-label:hover { border-color: #e94560; background: rgba(233,69,96,.12); }
.btn {
    padding: 10px 18px; border-radius: 10px; font-weight: 500; cursor: pointer;
    border: none; font-family: inherit; font-size: 13px; text-decoration: none;
    display: inline-flex; align-items: center; gap: 6px; color: #e8e5df;
}
.btn-sec { background: #0f3460; border: 1px solid rgba(232,229,223,.12); }
.btn-sec:hover { border-color: #53a8b6; }
.vazio {
    text-align: center; padding: 40px 20px; color: #9a978f;
    font-size: 13px; border: 1px dashed rgba(232,229,223,.12); border-radius: 10px;
}
.meta-grid {
    display: grid; grid-template-columns: repeat(auto-fit, minmax(150px, 1fr)); gap: 10px;
}
.meta-item {
    background: rgba(0,0,0,.15); border: 1px solid rgba(232,229,223,.1);
    border-radius: 10px; padding: 12px;
}
.meta-label { font-size: 10px; color: #9a978f; text-transform: uppercase; letter-spacing: .08em; }
.meta-valor { font-size: 14px; font-weight: 600; margin-top: 2px; word-break: break-word; }
.tag {
    display: inline-block; padding: 2px 10px; border-radius: 20px;
    font-size: 10px; font-weight: 600; text-transform: uppercase; letter-spacing: .05em;
}
.tag-gretl { background: rgba(83,168,182,.14); color: #53a8b6; border: 1px solid #53a8b6; }
.tag-gen { background: rgba(217,119,6,.14); color: #d97706; border: 1px solid #d97706; }
table { width: 100%; border-collapse: collapse; font-size: 13px; }
th, td { padding: 9px 10px; text-align: left; border-bottom: 1px solid rgba(232,229,223,.08); }
th { background: rgba(0,0,0,.2); font-size: 11px; text-transform: uppercase; color: #9a978f; letter-spacing: .05em; }
tr:hover td { background: rgba(255,255,255,.02); }
.table-wrap { overflow-x: auto; border: 1px solid rgba(232,229,223,.1); border-radius: 10px; }
.disc { background: rgba(13,150,105,.14); color: #0d9669; border: 1px solid #0d9669;
    padding: 1px 7px; border-radius: 10px; font-size: 10px; font-weight: 600; text-transform: uppercase; }
.cont { color: #9a978f; font-size: 11px; }
.xml-view {
    background: #0d1117; border: 1px solid rgba(232,229,223,.1); border-radius: 10px;
    padding: 14px; overflow-x: auto; font-family: monospace; font-size: 12px;
    line-height: 1.7; max-height: 320px; overflow-y: auto; white-space: pre;
}
.estat-grid {
    display: grid; grid-template-columns: repeat(auto-fit, minmax(180px, 1fr)); gap: 10px;
}
.estat-card {
    background: rgba(0,0,0,.15); border: 1px solid rgba(232,229,223,.1);
    border-radius: 10px; padding: 12px;
}
.estat-nome { font-weight: 600; color: #53a8b6; font-size: 13px; margin-bottom: 6px; }
.estat-label { font-size: 10px; color: #9a978f; margin-bottom: 8px; }
.estat-row { display: flex; justify-content: space-between; font-size: 12px; padding: 1px 0; }
.estat-campo { color: #9a978f; }
.estat-valor { font-weight: 500; font-variant-numeric: tabular-nums; }
.desc {
    background: #0f3460; border-left: 3px solid #53a8b6;
    padding: 10px 14px; border-radius: 0 8px 8px 0; font-size: 13px;
}
.icon {
    width: 32px; height: 32px; border-radius: 8px; display: inline-flex;
    align-items: center; justify-content: center; font-size: 16px;
}
.footer { text-align: center; color: #9a978f; font-size: 11px; padding: 16px; }
</style>
</head>
<body>
<div class="container">
    <div class="header">
        <h1><span class="gradient">GDT XML</span> Analisador Web</h1>
        <p class="sub">Carregue e analise arquivos GDT (Gretl Data Format) em XML</p>
    </div>
    
    <!-- CONTEUDO -->
    __CONTEUDO__
    
    <div class="footer">Servidor Python • Processamento local</div>
</div>
</body>
</html>"""

# ============================================================
# HANDLER
# ============================================================
class GDTHandler(BaseHTTPRequestHandler):
    
    def pagina_inicial(self):
        conteudo = '''
        <div class="card">
            <h2><span class="icon" style="background:rgba(233,69,96,.12);color:#e94560;">📁</span>Carregar Arquivo GDT.XML</h2>
            <form method="POST" action="/carregar" enctype="multipart/form-data" class="upload-area">
                <label class="file-label">
                    <input type="file" name="arquivo" accept=".gdt,.xml,text/xml" onchange="this.form.submit()">
                    📄 Selecionar arquivo
                </label>
                <a href="/exemplo" class="btn btn-sec" download>⬇ Baixar exemplo</a>
                <button type="button" class="btn btn-sec" onclick="usarExemplo()">⚡ Usar exemplo</button>
            </form>
            <p style="color:#9a978f;font-size:11px;margin-top:10px;">Formatos: .gdt / .xml</p>
        </div>
        <div class="card">
            <div class="vazio">
                <p>Nenhum arquivo carregado ainda.<br>Selecione um arquivo ou use o exemplo.</p>
            </div>
        </div>
        <script>
        function usarExemplo() {
            fetch("/exemplo_xml")
                .then(r => r.text())
                .then(xml => {
                    const blob = new Blob([xml], {type: "application/xml"});
                    const file = new File([blob], "exemplo.gdt.xml", {type: "application/xml"});
                    const dt = new DataTransfer();
                    dt.items.add(file);
                    const form = document.createElement("form");
                    form.method = "POST";
                    form.action = "/carregar";
                    form.enctype = "multipart/form-data";
                    const inp = document.createElement("input");
                    inp.type = "file";
                    inp.name = "arquivo";
                    inp.files = dt.files;
                    form.appendChild(inp);
                    document.body.appendChild(form);
                    form.submit();
                });
        }
        </script>'''
        return HTML_BASE.replace('__CONTEUDO__', conteudo)
    
    def pagina_resultado(self, xml_content, filename):
        resultado = analisar_gdt(xml_content)
        xml_formatado = formatar_xml(xml_content)
        
        if resultado.get('erro'):
            conteudo = f'''
            <div class="card">
                <h2><span class="icon" style="background:rgba(220,38,38,.12);color:#dc2626;">⚠</span>Erro na Análise</h2>
                <p style="color:#e8e5df;">{resultado['erro']}</p>
                <a href="/" class="btn btn-sec" style="margin-top:16px;">← Voltar</a>
            </div>'''
            return HTML_BASE.replace('__CONTEUDO__', conteudo)
        
        # Tag de tipo
        tag = '<span class="tag tag-gretl">Gretl GDT</span>' if resultado['eh_gretl'] else '<span class="tag tag-gen">XML Genérico</span>'
        arquivo_nome = f' • <span style="color:#9a978f;font-size:13px;">{filename}</span>' if filename else ''
        
        # Metadados
        labels_meta = {
            'version': 'Versão GDT', 'name': 'Nome Dataset',
            'frequency': 'Frequência', 'startobs': 'Obs. Inicial',
            'endobs': 'Obs. Final', 'type': 'Tipo Dados',
            'qtd_variaveis': 'Qtd. Variáveis', 'qtd_observacoes': 'Qtd. Observações',
            'tag_raiz': 'Tag Raiz'
        }
        meta_html = ''
        for chave, valor in resultado['metadados'].items():
            lbl = labels_meta.get(chave, chave)
            meta_html += f'<div class="meta-item"><div class="meta-label">{lbl}</div><div class="meta-valor">{valor}</div></div>'
        
        # Descrição
        desc_html = ''
        if resultado.get('descricao'):
            desc_html = f'<div class="desc">{resultado["descricao"]}</div>'
        
        # Variáveis
        vars_html = ''
        if resultado['variaveis']:
            linhas = ''
            for i, v in enumerate(resultado['variaveis']):
                tipo = '<span class="disc">Discreta</span>' if v['discreta'] else '<span class="cont">Contínua</span>'
                linhas += f'<tr><td style="color:#53a8b6;font-weight:600;">{i+1}</td><td style="font-family:monospace;font-weight:600;">{v["nome"]}</td><td>{v["label"]}</td><td>{tipo}</td></tr>'
            vars_html = f'''
            <h3>Variáveis ({len(resultado["variaveis"])})</h3>
            <div class="table-wrap">
                <table><thead><tr><th>#</th><th>Nome</th><th>Descrição</th><th>Tipo</th></tr></thead>
                <tbody>{linhas}</tbody></table>
            </div>'''
        
        # Observações
        obs_html = ''
        if resultado['observacoes'] and resultado['eh_gretl']:
            max_o = min(15, len(resultado['observacoes']))
            cab = ''.join(f'<th>{v["nome"]}</th>' for v in resultado['variaveis'])
            linhas = ''
            for i in range(max_o):
                cel = ''.join(f'<td style="font-family:monospace;">{val}</td>' for val in resultado['observacoes'][i])
                linhas += f'<tr><td style="color:#9a978f;">{i+1}</td>{cel}</tr>'
            total = len(resultado['observacoes'])
            aviso = f'<p style="color:#9a978f;font-size:11px;margin-top:6px;">Mostrando {max_o} de {total}</p>' if total > max_o else ''
            obs_html = f'<h3>Observações</h3><div class="table-wrap"><table><thead><tr><th>#</th>{cab}</tr></thead><tbody>{linhas}</tbody></table></div>{aviso}'
        
        # Estatísticas
        est_html = ''
        if resultado['estatisticas']:
            cards = ''
            for nome, s in resultado['estatisticas'].items():
                label = nome
                for v in resultado['variaveis']:
                    if v['nome'] == nome:
                        label = v['label'] or nome
                        break
                cards += f'''
                <div class="estat-card">
                    <div class="estat-nome">{nome}</div>
                    <div class="estat-label">{label}</div>
                    <div class="estat-row"><span class="estat-campo">Média</span><span class="estat-valor">{s["media"]}</span></div>
                    <div class="estat-row"><span class="estat-campo">Mediana</span><span class="estat-valor">{s["mediana"]}</span></div>
                    <div class="estat-row"><span class="estat-campo">Mín</span><span class="estat-valor">{s["min"]}</span></div>
                    <div class="estat-row"><span class="estat-campo">Máx</span><span class="estat-valor">{s["max"]}</span></div>
                    <div class="estat-row"><span class="estat-campo">Desvio</span><span class="estat-valor">{s["desvio"]}</span></div>
                    <div class="estat-row"><span class="estat-campo">N</span><span class="estat-valor">{s["n"]}</span></div>
                </div>'''
            est_html = f'<h3>Estatísticas Descritivas</h3><div class="estat-grid">{cards}</div>'
        
        # XML
        xml_vis = f'<h3>XML Completo</h3><div class="xml-view">{escape_html(xml_formatado)}</div>'
        
        conteudo = f'''
        <div class="card">
            <h2><span class="icon" style="background:rgba(83,168,182,.12);color:#53a8b6;">📊</span>
            Resultados da Análise {tag}{arquivo_nome}</h2>
            {desc_html}
            <h3>Metadados</h3>
            <div class="meta-grid">{meta_html}</div>
            {vars_html}
            {obs_html}
            {est_html}
            {xml_vis}
            <a href="/" class="btn btn-sec" style="margin-top:20px;">← Carregar outro arquivo</a>
        </div>'''
        
        return HTML_BASE.replace('__CONTEUDO__', conteudo)
    
    def do_GET(self):
        try:
            if self.path == '/':
                self.send_response(200)
                self.send_header('Content-Type', 'text/html; charset=utf-8')
                self.end_headers()
                self.wfile.write(self.pagina_inicial().encode('utf-8'))
            
            elif self.path == '/exemplo':
                self.send_response(200)
                self.send_header('Content-Type', 'application/xml')
                self.send_header('Content-Disposition', 'attachment; filename="exemplo.gdt.xml"')
                self.end_headers()
                self.wfile.write(SAMPLE_GDT.encode('utf-8'))
            
            elif self.path == '/exemplo_xml':
                self.send_response(200)
                self.send_header('Content-Type', 'application/xml; charset=utf-8')
                self.end_headers()
                self.wfile.write(SAMPLE_GDT.encode('utf-8'))
            
            else:
                self.send_response(404)
                self.end_headers()
                self.wfile.write(b'404 - Nao encontrado')
        except Exception as e:
            print(f"ERRO no GET: {e}")
            self.send_response(500)
            self.end_headers()
    
    def do_POST(self):
        try:
            if self.path == '/carregar':
                content_type = self.headers.get('Content-Type', '')
                
                if 'multipart/form-data' in content_type:
                    boundary = content_type.split('boundary=')[1].encode()
                    content_length = int(self.headers['Content-Length'])
                    post_data = self.rfile.read(content_length)
                    
                    parts = post_data.split(b'--' + boundary)
                    xml_content = None
                    filename = 'arquivo.gdt.xml'
                    
                    for part in parts:
                        if b'filename=' in part:
                            try:
                                fn = part.split(b'filename="')[1].split(b'"')[0]
                                filename = fn.decode('utf-8', errors='ignore')
                            except:
                                pass
                            header_end = part.find(b'\r\n\r\n')
                            if header_end != -1:
                                conteudo = part[header_end+4:].rstrip(b'\r\n--')
                                xml_content = conteudo.decode('utf-8', errors='ignore')
                                break
                    
                    if xml_content and xml_content.strip():
                        self.send_response(200)
                        self.send_header('Content-Type', 'text/html; charset=utf-8')
                        self.end_headers()
                        self.wfile.write(self.pagina_resultado(xml_content, filename).encode('utf-8'))
                    else:
                        self.send_response(200)
                        self.send_header('Content-Type', 'text/html; charset=utf-8')
                        self.end_headers()
                        msg = '<div class="card"><h2>⚠ Arquivo vazio ou inválido</h2><a href="/" class="btn btn-sec">Voltar</a></div>'
                        self.wfile.write(HTML_BASE.replace('__CONTEUDO__', msg).encode('utf-8'))
                else:
                    self.do_GET()
        except Exception as e:
            print(f"ERRO no POST: {e}")
            import traceback
            traceback.print_exc()
            self.send_response(500)
            self.end_headers()
    
    def log_message(self, format, *args):
        print(f"[GDT] {args[0]}")

# ============================================================
# MAIN
# ============================================================
def main():
    porta = 8000
    try:
        servidor = HTTPServer(('0.0.0.0', porta), GDTHandler)
        print("=" * 50)
        print("  SERVIDOR GDT XML - PRONTO")
        print("=" * 50)
        print(f"  Acesse: http://localhost:{porta}")
        print(f"  Ou: http://127.0.0.1:{porta}")
        print("  Ctrl+C para parar")
        print("=" * 50)
        servidor.serve_forever()
    except KeyboardInterrupt:
        print("\nServidor parado.")
    except Exception as e:
        print(f"ERRO ao iniciar: {e}")
        print("Tente outra porta: altere a variavel 'porta' no final do arquivo")

if __name__ == '__main__':
    main()
