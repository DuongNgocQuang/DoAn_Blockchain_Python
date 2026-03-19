[app_blockchain.py](https://github.com/user-attachments/files/26105978/app_blockchain.py)
from flask import Flask, jsonify, request, render_template_string
import hashlib
import time

# --- 1. PHẦN LÕI: THUẬT TOÁN BLOCKCHAIN BẢO MẬT CAO ---
class BankBlockchain:
    def __init__(self):
        self.chain = []
        self.current_transactions = [] 
        self.create_block(proof=1, previous_hash='0') 

    def new_transaction(self, sender, receiver, amount):
        self.current_transactions.append({
            'sender': sender,      
            'receiver': receiver,  
            'amount': amount,
            'timestamp': time.strftime("%H:%M:%S %d/%m/%Y", time.localtime()) # Thêm giờ giao dịch thực tế
        })
        return self.get_previous_block()['index'] + 1

    def create_block(self, proof, previous_hash):
        block = {
            'index': len(self.chain) + 1,
            'timestamp': time.strftime("%H:%M:%S %d/%m/%Y", time.localtime()),
            'transactions': self.current_transactions, 
            'proof': proof,
            'previous_hash': previous_hash
        }
        self.current_transactions = [] 
        self.chain.append(block)
        return block

    def get_previous_block(self):
        return self.chain[-1]

    def proof_of_work(self, previous_proof):
        new_proof = 1
        check_proof = False
        while check_proof is False:
            hash_operation = hashlib.sha256(str(new_proof**2 - previous_proof**2).encode()).hexdigest()
            if hash_operation[:4] == '0000':
                check_proof = True
            else:
                new_proof += 1
        return new_proof

    def hash(self, block):
        encoded_block = str(block).encode()
        return hashlib.sha256(encoded_block).hexdigest()

    def is_chain_valid(self, chain):
        # Tính năng quét toàn vẹn chuỗi
        previous_block = chain[0]
        block_index = 1
        while block_index < len(chain):
            block = chain[block_index]
            if block['previous_hash'] != self.hash(previous_block):
                return False
            previous_proof = previous_block['proof']
            proof = block['proof']
            hash_operation = hashlib.sha256(str(proof**2 - previous_proof**2).encode()).hexdigest()
            if hash_operation[:4] != '0000':
                return False
            previous_block = block
            block_index += 1
        return True

# --- 2. PHẦN MÁY CHỦ WEB & GIAO DIỆN CỰC KHỦNG ---
app = Flask(__name__)
blockchain = BankBlockchain()

HTML_TEMPLATE = """
<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <title>Hệ Thống Blockchain Ngân Hàng Core</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <style> 
        body { background-color: #0d1117; color: #c9d1d9; font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; }
        .card { background-color: #161b22; border: 1px solid #30363d; color: #c9d1d9; }
        .block-card { border-left: 5px solid #58a6ff; margin-bottom: 15px; background-color: #0d1117; border-radius: 8px;} 
        .form-control { background-color: #0d1117; color: white; border: 1px solid #30363d; }
        .form-control::placeholder { color: #8b949e; }
        .form-control:focus { background-color: #0d1117; color: white; border-color: #58a6ff; box-shadow: none;}
        .coin-price { font-size: 1.5rem; font-weight: bold; text-shadow: 0 0 10px rgba(88, 166, 255, 0.5); }
        
        /* Màn hình đăng nhập */
        #login-screen { display: flex; align-items: center; justify-content: center; height: 100vh; background-color: #0d1117; }
        .login-box { width: 100%; max-width: 400px; padding: 40px; border-radius: 10px; background-color: #161b22; box-shadow: 0 0 20px rgba(0,0,0,0.5); }
        
        /* Ẩn phần mềm chính lúc đầu */
        #main-app { display: none; }
    </style>
</head>
<body>

    <div id="login-screen">
        <div class="login-box text-center border border-secondary">
            <h3 class="text-primary mb-4">🔐 ĐĂNG NHẬP HỆ THỐNG BLOCKCHAIN</h3>
            <div class="mb-3"><input type="text" id="username" class="form-control" placeholder="Tài khoản (Nhập: admin)"></div>
            <div class="mb-4"><input type="password" id="password" class="form-control" placeholder="Mật khẩu (Nhập: 123456)"></div>
            <button class="btn btn-primary w-100 fw-bold" onclick="login()">TRUY CẬP SỔ CÁI</button>
            <p id="login-error" class="text-danger mt-3" style="display:none;">Sai tài khoản hoặc mật khẩu!</p>
        </div>
    </div>

    <div id="main-app" class="container mt-4">
        <div class="d-flex justify-content-between align-items-center mb-4">
            <h2 class="text-primary m-0">🌐 NODE QUẢN TRỊ BLOCKCHAIN NGÂN HÀNG</h2>
            <button class="btn btn-outline-danger btn-sm" onclick="logout()">Đăng xuất</button>
        </div>
        
        <div class="row mb-4" id="crypto-prices">
            <div class="col-12 text-center text-warning">Đang kết nối API thị trường toàn cầu...</div>
        </div>

        <div class="row">
            <div class="col-md-4">
                <div class="card p-4 shadow-sm mb-4">
                    <h4 class="text-warning border-bottom border-warning pb-2">Tạo Lệnh Chuyển Tiền</h4>
                    <div class="mb-2"><input type="text" id="sender" class="form-control" placeholder="Tài khoản gửi (VD: VCB)"></div>
                    <div class="mb-2"><input type="text" id="receiver" class="form-control" placeholder="Tài khoản nhận (VD: BIDV)"></div>
                    <div class="mb-3"><input type="number" id="amount" class="form-control" placeholder="Số tiền (VNĐ)"></div>
                    <button class="btn btn-warning w-100 fw-bold text-dark" onclick="addTransaction()">ĐẨY VÀO HÀNG ĐỢI (MEMPOOL)</button>
                    <p id="tx-msg" class="text-success mt-2 fw-bold text-center"></p>
                </div>
            </div>

            <div class="col-md-8">
                <div class="card p-4 shadow-sm mb-4">
                    <h4 class="text-info border-bottom border-info pb-2">Bảng Điều Khiển Hệ Thống Lõi</h4>
                    <div class="d-flex gap-2 mb-3">
                        <button class="btn btn-danger w-100 fw-bold" onclick="mineBlock()">⛏️ ĐÀO KHỐI (PROOF OF WORK)</button>
                        <button class="btn btn-info w-100 fw-bold text-white" onclick="getChain()">📜 XUẤT SỔ CÁI</button>
                    </div>
                    <button class="btn btn-success w-100 fw-bold" onclick="validateChain()">🛡️ QUÉT BẢO MẬT & KIỂM TRA TOÀN VẸN CHUỖI</button>
                </div>
                
                <div id="result-screen" class="p-4 border rounded" style="background-color: #010409; border-color: #30363d !important; min-height: 300px; max-height: 600px; overflow-y: auto;">
                    <div class="text-center text-secondary mt-5">
                        <h5>Hệ thống đã sẵn sàng.</h5>
                        <p>Nhập giao dịch và tiến hành đào khối để bắt đầu.</p>
                    </div>
                </div>
            </div>
        </div>
    </div>

    <script>
        // --- HỆ THỐNG ĐĂNG NHẬP GIẢ LẬP ---
        function login() {
            let u = document.getElementById('username').value;
            let p = document.getElementById('password').value;
            if(u === 'admin' && p === '123456') {
                document.getElementById('login-screen').style.display = 'none';
                document.getElementById('main-app').style.display = 'block';
                fetchPrices(); // Bắt đầu load giá khi đăng nhập thành công
            } else {
                document.getElementById('login-error').style.display = 'block';
            }
        }

        function logout() {
            document.getElementById('main-app').style.display = 'none';
            document.getElementById('login-screen').style.display = 'flex';
            document.getElementById('username').value = '';
            document.getElementById('password').value = '';
            document.getElementById('login-error').style.display = 'none';
        }

        // --- HÚT GIÁ COIN TỪ API QUỐC TẾ ---
        async function fetchPrices() {
            try {
                let res = await fetch('https://api.coingecko.com/api/v3/simple/price?ids=bitcoin,ethereum,binancecoin,solana&vs_currencies=usd');
                let data = await res.json();
                let html = `
                    <div class="col-md-3"><div class="card p-2 shadow text-center border-bottom border-warning"><span class="text-secondary small">Bitcoin (BTC)</span><div class="coin-price text-warning">$${data.bitcoin.usd.toLocaleString()}</div></div></div>
                    <div class="col-md-3"><div class="card p-2 shadow text-center border-bottom border-primary"><span class="text-secondary small">Ethereum (ETH)</span><div class="coin-price text-primary">$${data.ethereum.usd.toLocaleString()}</div></div></div>
                    <div class="col-md-3"><div class="card p-2 shadow text-center border-bottom border-warning"><span class="text-secondary small">BNB</span><div class="coin-price text-warning">$${data.binancecoin.usd.toLocaleString()}</div></div></div>
                    <div class="col-md-3"><div class="card p-2 shadow text-center border-bottom border-success"><span class="text-secondary small">Solana (SOL)</span><div class="coin-price text-success">$${data.solana.usd.toLocaleString()}</div></div></div>
                `;
                document.getElementById('crypto-prices').innerHTML = html;
            } catch (error) { console.log("Lỗi API Giá"); }
        }
        setInterval(fetchPrices, 15000);

        // --- CÁC HÀM TƯƠNG TÁC API BLOCKCHAIN LÕI ---
        function addTransaction() {
            let sender = document.getElementById('sender').value;
            let receiver = document.getElementById('receiver').value;
            let amount = document.getElementById('amount').value;
            if(!sender || !receiver || !amount) { alert("Vui lòng điền đủ thông tin!"); return; }

            fetch('/add_transaction', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ "sender": sender, "receiver": receiver, "amount": amount })
            })
            .then(res => res.json())
            .then(data => { 
                document.getElementById('tx-msg').innerText = "✔️ " + data.message; 
                setTimeout(() => { document.getElementById('tx-msg').innerText = ""; }, 3000);
            });
        }

        function mineBlock() {
            // Hiệu ứng Spinner quay quay cực xịn
            document.getElementById('result-screen').innerHTML = `
                <div class="text-center mt-5">
                    <div class="spinner-border text-danger" style="width: 3rem; height: 3rem;" role="status"></div>
                    <h5 class="text-danger mt-3 blink">Đang vắt kiệt CPU để giải mã SHA-256...</h5>
                    <p class="text-secondary">Đang tìm số Nonce hợp lệ...</p>
                </div>`;
            
            // Cố tình delay 2 giây để tạo cảm giác máy tính đang phải tính toán nặng
            setTimeout(() => {
                fetch('/mine_block')
                .then(res => res.json())
                .then(data => {
                    let html = `<h4 class='text-success mb-3'>✔️ ${data.message}</h4>`;
                    html += `<div class="p-3 border border-success rounded bg-dark">
                                <p class="text-light mb-1"><b>Khối số:</b> <span class="badge bg-primary fs-6">${data.index}</span></p>
                                <p class="text-light mb-1"><b>Bằng chứng (Nonce):</b> ${data.proof}</p>
                                <p class="text-light mb-0"><b>Mã băm (Hash) rễ:</b> <span class='text-warning' style="word-break: break-all;">${data.previous_hash}</span></p>
                             </div>`;
                    document.getElementById('result-screen').innerHTML = html;
                });
            }, 2000);
        }

        function getChain() {
            fetch('/get_chain')
            .then(res => res.json())
            .then(data => {
                let html = `<h4 class='text-info mb-4'>📜 Sổ Cái Toàn Cầu (${data.length} Khối)</h4>`;
                data.chain.forEach(block => {
                    // Hiển thị chi tiết giao dịch (Tính năng mới)
                    let txHtml = "";
                    if (block.transactions.length === 0) {
                        txHtml = "<i class='text-secondary'>Không có giao dịch (Khối nguyên thủy)</i>";
                    } else {
                        block.transactions.forEach((tx, idx) => {
                            txHtml += `<div class="bg-dark p-2 rounded mt-2 border border-secondary" style="font-size: 0.9rem;">
                                        <span class="text-info">[${tx.timestamp}]</span> 
                                        <b>${tx.sender}</b> ➡️ <b>${tx.receiver}</b>: 
                                        <span class="text-warning fw-bold">${Number(tx.amount).toLocaleString()} VNĐ</span>
                                       </div>`;
                        });
                    }

                    html += `<div class='card text-light block-card p-3 mb-3 border border-secondary'>
                        <div class="d-flex justify-content-between border-bottom border-secondary pb-2 mb-2">
                            <b class="text-primary fs-5">Khối #${block.index}</b>
                            <span class="text-muted">${block.timestamp}</span>
                        </div>
                        <p class="mb-1 text-muted" style="font-size:0.85rem;"><b>Mã liên kết:</b> ${block.previous_hash}</p>
                        <div class="mt-2">
                            <b>📑 Biên lai giao dịch:</b>
                            ${txHtml}
                        </div>
                    </div>`;
                });
                document.getElementById('result-screen').innerHTML = html;
            });
        }

        function validateChain() {
            // Nút kiểm tra toàn vẹn chuỗi (Tính năng mới)
            document.getElementById('result-screen').innerHTML = `
                <div class="text-center mt-5">
                    <div class="spinner-grow text-success" style="width: 3rem; height: 3rem;" role="status"></div>
                    <h5 class="text-success mt-3">Đang rà soát toàn bộ sổ cái...</h5>
                </div>`;
                
            setTimeout(() => {
                fetch('/validate_chain')
                .then(res => res.json())
                .then(data => {
                    if(data.valid) {
                        document.getElementById('result-screen').innerHTML = `
                            <div class="alert alert-success text-center mt-4 border border-success">
                                <h3>✅ AN TOÀN TUYỆT ĐỐI</h3>
                                <p class="mb-0 fs-5">${data.message}</p>
                            </div>`;
                    } else {
                        document.getElementById('result-screen').innerHTML = `
                            <div class="alert alert-danger text-center mt-4 border border-danger">
                                <h3>❌ BÁO ĐỘNG ĐỎ</h3>
                                <p class="mb-0 fs-5">${data.message}</p>
                            </div>`;
                    }
                });
            }, 1500);
        }
    </script>
</body>
</html>
"""

# Các đường dẫn kết nối (API Routes)
@app.route('/', methods=['GET'])
def home():
    return render_template_string(HTML_TEMPLATE)

@app.route('/mine_block', methods=['GET'])
def mine_block():
    previous_block = blockchain.get_previous_block()
    previous_proof = previous_block['proof']
    proof = blockchain.proof_of_work(previous_proof)
    previous_hash = blockchain.hash(previous_block)
    block = blockchain.create_block(proof, previous_hash)
    
    response = {
        'message': 'ĐÃ KHAI THÁC THÀNH CÔNG KHỐI MỚI!',
        'index': block['index'],
        'transactions': block['transactions'],
        'proof': block['proof'],
        'previous_hash': block['previous_hash']
    }
    return jsonify(response), 200

@app.route('/get_chain', methods=['GET'])
def get_chain():
    return jsonify({'chain': blockchain.chain, 'length': len(blockchain.chain)}), 200

@app.route('/add_transaction', methods=['POST'])
def add_transaction():
    json_data = request.get_json()
    index = blockchain.new_transaction(json_data['sender'], json_data['receiver'], json_data['amount'])
    return jsonify({'message': f'Giao dịch được đưa vào Khối số {index}'}), 201

@app.route('/validate_chain', methods=['GET'])
def validate_chain():
    is_valid = blockchain.is_chain_valid(blockchain.chain)
    if is_valid:
        return jsonify({'message': 'Mạng lưới HỢP LỆ. Tất cả các khối đều liên kết chặt chẽ và không bị chỉnh sửa.', 'valid': True}), 200
    else:
        return jsonify({'message': 'MẠNG LƯỚI BỊ XÂM PHẠM! Có dữ liệu đã bị chỉnh sửa trái phép.', 'valid': False}), 200

if __name__ == '__main__':
    app.run(host='127.0.0.1', port=5000)
