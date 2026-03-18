import hashlib
import time

class Blockchain:
    def __init__(self):
        self.chain = []
        # Tạo khối nguyên thủy (Genesis block)
        self.create_block(proof=1, previous_hash='0', data="Khối khởi nguồn")

    def create_block(self, proof, previous_hash, data):
        block = {
            'index': len(self.chain) + 1,
            'timestamp': time.time(),
            'data': data,
            'proof': proof, # Số Nonce (Bằng chứng công việc)
            'previous_hash': previous_hash
        }
        self.chain.append(block)
        return block

    def get_previous_block(self):
        return self.chain[-1]

    def proof_of_work(self, previous_proof):
        new_proof = 1
        check_proof = False
        # Vòng lặp liên tục thử số mới cho đến khi giải được bài toán
        while check_proof is False:
            # Bài toán: Mã băm của (new_proof^2 - previous_proof^2) phải bắt đầu bằng '0000'
            hash_operation = hashlib.sha256(str(new_proof**2 - previous_proof**2).encode()).hexdigest()
            if hash_operation[:4] == '0000':
                check_proof = True
            else:
                new_proof += 1
        return new_proof

    def hash(self, block):
        encoded_block = str(block).encode()
        return hashlib.sha256(encoded_block).hexdigest()

# Kịch bản chạy thử nghiệm
if __name__ == '__main__':
    my_coin = Blockchain()
    print("Mạng Blockchain đã khởi động với khối đầu tiên!")
    
    # Tiến hành đào thêm 2 khối mới lưu trữ dữ liệu
    for i in range(2):
        previous_block = my_coin.get_previous_block()
        previous_proof = previous_block['proof']
        
        print(f"\nĐang giải bài toán PoW cho khối thứ {len(my_coin.chain) + 1}...")
        proof = my_coin.proof_of_work(previous_proof)
        
        previous_hash = my_coin.hash(previous_block)
        data = f"Giao dịch số {i+1}: Sinh viên A chuyển dữ liệu cho Sinh viên B"
        block = my_coin.create_block(proof, previous_hash, data)
        
        print("=> Đào thành công!")
        print(f"Index: {block['index']}")
        print(f"Data: {block['data']}")
        print(f"Proof (Nonce): {block['proof']}")
        print(f"Hash của khối trước: {block['previous_hash']}")
