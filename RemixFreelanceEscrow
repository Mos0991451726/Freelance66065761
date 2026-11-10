// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title FreelanceEscrow
 * @notice Freelance Escrow & Smart Payment Platform (DigitalWork Solutions)
 *
 * วงจรสถานะของงาน (State machine):
 *
 *   Created --deposit--> Funded --submitWork--> Submitted --approve--> Completed
 *      |                   |  \                    |
 *      |                   |   \--(เลย deadline,   +--raiseDispute--> Disputed --resolve--> Completed | Refunded
 *      |                   |      ไม่ส่งงาน)--> Refunded (ผู้ว่าจ้างเรียกเงินคืนเอง)
 *      |                   +--raiseDispute--> Disputed
 *      +--cancel--> Cancelled (ยังไม่ฝากเงิน)
 *
 * กฎทางธุรกิจ:
 *  1) ห้ามจ่ายเงินซ้ำ        : เงินออกได้เมื่อสถานะเปลี่ยนเป็น Completed/Refunded เท่านั้น
 *                              และสถานะเหล่านี้เป็นสถานะปลายทาง (เปลี่ยนต่อไม่ได้)
 *  2) เบิกเงินจาก Escrow ไม่ได้ถ้ายังไม่เข้าเงื่อนไข : ไม่มีฟังก์ชันถอนอิสระ
 *                              เงินออกผ่าน 4 เส้นทางที่ตรวจเงื่อนไขเท่านั้น
 *                              (approve / resolveDispute / refundAfterDeadline / ไม่มีทางอื่น)
 */
contract FreelanceEscrow {
    // ---------------------------------------------------------------- Types
    enum Status {
        Created,    // 0 สร้างข้อตกลงแล้ว รอฝากเงิน
        Funded,     // 1 ฝากเงินแล้ว รอส่งงาน
        Submitted,  // 2 ส่งมอบงานแล้ว รอผู้ว่าจ้างตรวจรับ
        Disputed,   // 3 มีข้อพิพาท รอผู้ดูแลตัดสิน
        Completed,  // 4 จ่ายเงินให้ฟรีแลนซ์แล้ว
        Refunded,   // 5 คืนเงินให้ผู้ว่าจ้างแล้ว
        Cancelled   // 6 ยกเลิกก่อนฝากเงิน
    }

    struct Job {
        uint256 id;
        address client;              // ผู้ว่าจ้าง
        address payable freelancer;  // ฟรีแลนซ์
        string title;                // ชื่อโครงการ
        string description;          // รายละเอียดงาน
        uint256 amount;              // ค่าจ้าง (wei)
        uint256 deadline;            // กำหนดส่ง (unix timestamp)
        string deliverable;          // หลักฐานส่งมอบงาน (URL / Hash)
        Status status;
        address disputeRaisedBy;     // ผู้แจ้งข้อพิพาท
        uint256 createdAt;
        uint256 updatedAt;
    }

    // -------------------------------------------------------------- Storage
    address public admin;            // ผู้ดูแลที่ได้รับสิทธิ์ตัดสินข้อพิพาท
    uint256 public jobCount;
    mapping(uint256 => Job) private jobs;
    mapping(address => uint256[]) private jobsOf; // jobId ของแต่ละกระเป๋า (ทั้งฐานะผู้ว่าจ้าง/ฟรีแลนซ์)

    bool private locked; // reentrancy guard

    // --------------------------------------------------------------- Events
    // ใช้เป็น "ประวัติธุรกรรม" ให้หน้าเว็บค้นหา พร้อม Transaction Hash
    event JobCreated(uint256 indexed jobId, address indexed client, address indexed freelancer, uint256 amount, uint256 deadline);
    event PaymentDeposited(uint256 indexed jobId, address indexed client, uint256 amount);
    event WorkSubmitted(uint256 indexed jobId, address indexed freelancer, string deliverable);
    event PaymentReleased(uint256 indexed jobId, address indexed freelancer, uint256 amount);
    event DisputeRaised(uint256 indexed jobId, address indexed raisedBy, string reason);
    event DisputeResolved(uint256 indexed jobId, address indexed resolvedBy, bool paidToFreelancer, uint256 amount);
    event PaymentRefunded(uint256 indexed jobId, address indexed client, uint256 amount);
    event JobCancelled(uint256 indexed jobId, address indexed client);
    event AdminChanged(address indexed oldAdmin, address indexed newAdmin);

    // ------------------------------------------------------------ Modifiers
    modifier onlyAdmin() {
        require(msg.sender == admin, "Only admin");
        _;
    }

    modifier jobExists(uint256 jobId) {
        require(jobId > 0 && jobId <= jobCount, "Job does not exist");
        _;
    }

    modifier nonReentrant() {
        require(!locked, "Reentrancy");
        locked = true;
        _;
        locked = false;
    }

    // ---------------------------------------------------------- Constructor
    constructor() {
        admin = msg.sender; // ผู้ deploy เป็นผู้ดูแล
        emit AdminChanged(address(0), msg.sender);
    }

    // ============================================================ 1) CREATE
    /// @notice ผู้ว่าจ้างสร้างข้อตกลงจ้างงาน (ยังไม่ต้องจ่ายเงินในขั้นนี้)
    /// @param title        ชื่อโครงการ
    /// @param description  รายละเอียดงาน
    /// @param freelancer   Wallet Address ของฟรีแลนซ์
    /// @param amount       ค่าจ้างเป็น wei (1 ETH = 1e18 wei)
    /// @param deadline     กำหนดส่ง (unix timestamp วินาที) ต้องอยู่ในอนาคต
    function createJob(
        string calldata title,
        string calldata description,
        address payable freelancer,
        uint256 amount,
        uint256 deadline
    ) external returns (uint256 jobId) {
        require(bytes(title).length > 0, "Title required");
        require(freelancer != address(0), "Invalid freelancer");
        require(freelancer != msg.sender, "Client cannot be freelancer");
        require(freelancer != admin && msg.sender != admin, "Admin cannot be a party");
        require(amount > 0, "Amount must be > 0");
        require(deadline > block.timestamp, "Deadline must be in the future");

        jobId = ++jobCount;
        Job storage j = jobs[jobId];
        j.id = jobId;
        j.client = msg.sender;
        j.freelancer = freelancer;
        j.title = title;
        j.description = description;
        j.amount = amount;
        j.deadline = deadline;
        j.status = Status.Created;
        j.createdAt = block.timestamp;
        j.updatedAt = block.timestamp;

        jobsOf[msg.sender].push(jobId);
        jobsOf[freelancer].push(jobId);

        emit JobCreated(jobId, msg.sender, freelancer, amount, deadline);
    }

    /// @notice ผู้ว่าจ้างยกเลิกข้อตกลงที่ยังไม่ได้ฝากเงิน
    function cancelJob(uint256 jobId) external jobExists(jobId) {
        Job storage j = jobs[jobId];
        require(msg.sender == j.client, "Only client");
        require(j.status == Status.Created, "Cannot cancel after deposit");

        j.status = Status.Cancelled;
        j.updatedAt = block.timestamp;
        emit JobCancelled(jobId, msg.sender);
    }

    // =========================================================== 2) DEPOSIT
    /// @notice ผู้ว่าจ้างโอน ETH เข้า Smart Contract (ต้องเท่ากับค่าจ้างพอดี)
    function depositPayment(uint256 jobId) external payable jobExists(jobId) {
        Job storage j = jobs[jobId];
        require(msg.sender == j.client, "Only client");
        require(j.status == Status.Created, "Already funded or closed"); // กันฝากซ้ำ
        require(block.timestamp < j.deadline, "Deadline passed");
        require(msg.value == j.amount, "Must deposit the exact amount");

        j.status = Status.Funded;
        j.updatedAt = block.timestamp;
        emit PaymentDeposited(jobId, msg.sender, msg.value);
    }

    // ========================================================== 3) SUBMIT
    /// @notice ฟรีแลนซ์บันทึกหลักฐานการส่งมอบงาน (URL หรือ Hash ของไฟล์)
    function submitWork(uint256 jobId, string calldata deliverable) external jobExists(jobId) {
        Job storage j = jobs[jobId];
        require(msg.sender == j.freelancer, "Only freelancer");
        require(j.status == Status.Funded, "Job is not in Funded state");
        require(block.timestamp <= j.deadline, "Deadline passed");
        require(bytes(deliverable).length > 0, "Deliverable required");

        j.deliverable = deliverable;
        j.status = Status.Submitted;
        j.updatedAt = block.timestamp;
        emit WorkSubmitted(jobId, msg.sender, deliverable);
    }

    // =========================================================== 4) APPROVE
    /// @notice ผู้ว่าจ้างตรวจรับงาน -> โอน ETH ให้ฟรีแลนซ์โดยอัตโนมัติ
    function approveAndRelease(uint256 jobId) external jobExists(jobId) nonReentrant {
        Job storage j = jobs[jobId];
        require(msg.sender == j.client, "Only client");
        require(j.status == Status.Submitted, "Work not submitted");

        // Effects ก่อน Interactions (กันจ่ายซ้ำ / reentrancy)
        j.status = Status.Completed;
        j.updatedAt = block.timestamp;
        uint256 amount = j.amount;

        _send(j.freelancer, amount);
        emit PaymentReleased(jobId, j.freelancer, amount);
    }

    // =========================================================== 5) DISPUTE
    /// @notice ผู้ว่าจ้างหรือฟรีแลนซ์แจ้งข้อพิพาท (ล็อกเงินไว้รอผู้ดูแล)
    function raiseDispute(uint256 jobId, string calldata reason) external jobExists(jobId) {
        Job storage j = jobs[jobId];
        require(msg.sender == j.client || msg.sender == j.freelancer, "Only parties");
        require(j.status == Status.Funded || j.status == Status.Submitted, "Cannot dispute in this state");
        require(bytes(reason).length > 0, "Reason required");

        j.status = Status.Disputed;
        j.disputeRaisedBy = msg.sender;
        j.updatedAt = block.timestamp;
        emit DisputeRaised(jobId, msg.sender, reason);
    }

    /// @notice ผู้ดูแลพิจารณา: จ่ายให้ฟรีแลนซ์ หรือ คืนเงินผู้ว่าจ้าง
    function resolveDispute(uint256 jobId, bool payFreelancer) external jobExists(jobId) onlyAdmin nonReentrant {
        Job storage j = jobs[jobId];
        require(j.status == Status.Disputed, "No active dispute");
        require(msg.sender != j.client && msg.sender != j.freelancer, "Admin is a party");

        uint256 amount = j.amount;
        j.updatedAt = block.timestamp;

        if (payFreelancer) {
            j.status = Status.Completed;
            _send(j.freelancer, amount);
            emit PaymentReleased(jobId, j.freelancer, amount);
        } else {
            j.status = Status.Refunded;
            _send(payable(j.client), amount);
            emit PaymentRefunded(jobId, j.client, amount);
        }
        emit DisputeResolved(jobId, msg.sender, payFreelancer, amount);
    }

    /// @notice ผู้ว่าจ้างเรียกเงินคืนเองได้ ถ้าเลยกำหนดส่งแล้วฟรีแลนซ์ยังไม่ส่งงาน
    function refundAfterDeadline(uint256 jobId) external jobExists(jobId) nonReentrant {
        Job storage j = jobs[jobId];
        require(msg.sender == j.client, "Only client");
        require(j.status == Status.Funded, "Work already submitted or closed");
        require(block.timestamp > j.deadline, "Deadline not reached");

        j.status = Status.Refunded;
        j.updatedAt = block.timestamp;
        uint256 amount = j.amount;

        _send(payable(j.client), amount);
        emit PaymentRefunded(jobId, j.client, amount);
    }

    // ========================================================== 6) TRACKING
    /// @notice ดูรายละเอียดงานทั้งหมด
    function getJob(uint256 jobId) external view jobExists(jobId) returns (Job memory) {
        return jobs[jobId];
    }

    /// @notice รายการ jobId ที่กระเป๋านี้เกี่ยวข้อง (เป็นผู้ว่าจ้างหรือฟรีแลนซ์)
    function getJobIdsOf(address user) external view returns (uint256[] memory) {
        return jobsOf[user];
    }

    /// @notice เงินที่ถูกล็อกอยู่ในงานนี้ (0 ถ้ายังไม่ฝาก หรือจ่าย/คืนไปแล้ว)
    function escrowBalanceOf(uint256 jobId) public view jobExists(jobId) returns (uint256) {
        Status s = jobs[jobId].status;
        if (s == Status.Funded || s == Status.Submitted || s == Status.Disputed) {
            return jobs[jobId].amount;
        }
        return 0;
    }

    /// @notice ยอดรวม ETH ที่ถูกล็อกใน Escrow ทั้งสัญญา (ต้องเท่ากับ address(this).balance เสมอ)
    function totalLocked() external view returns (uint256) {
        return address(this).balance;
    }

    // ================================================================ Admin
    /// @notice โอนสิทธิ์ผู้ดูแลให้กระเป๋าอื่น
    function changeAdmin(address newAdmin) external onlyAdmin {
        require(newAdmin != address(0), "Invalid admin");
        emit AdminChanged(admin, newAdmin);
        admin = newAdmin;
    }

    // ============================================================== Internal
    function _send(address payable to, uint256 amount) private {
        (bool ok, ) = to.call{value: amount}("");
        require(ok, "ETH transfer failed");
    }

    // ไม่รับ ETH ที่โอนเข้ามาตรง ๆ (ต้องฝากผ่าน depositPayment เท่านั้น ไม่งั้นเงินจะค้างและไม่ตรงกับบัญชีงาน)
    receive() external payable {
        revert("Use depositPayment");
    }

    fallback() external payable {
        revert("Invalid call");
    }
}
