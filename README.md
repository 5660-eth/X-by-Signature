# X-by-Signature
Transfer and trade by Signature, token extention

摘要
通过签名转移和交易token，使得账户而不必拥有ETH作为gas。同时引入开始时间（StartTime），使得定期支付成为可能（如支付工资、房租）
payable at a fixed period after date
出票后定期付款

动机
使得二等代币（如ERC20，引自EIP2612）成为1.5等代币，提升用户使用钱包的体验。尽管有一些方案如4337，尝试在钱包层面提升用户体验，但在token层面的提升也是必要的。
EIP2612提供了一种方式，使得在本账户不存在ETH的情况，仍然可以通过签名完成一些交易。。。但通过授权的方式，会带来不可知的风险，如未预期的支出，单点故障等，这种方式是中心化的，受到审查等因素影响。此外，EIP2612也不支持合约账户多签。
增强一种对授权模式的替代（新想的，加入了时间因素，可能不用）

规格

基本原理
聚焦用户核心操作。尽管用户链上行为逐渐复杂，但对资产的行为仍可抽象为转移和交易，其他行为可以是这两个行为的组合，或者通过交易获取ETH主动完成。
通过引入小费概念，使得去中心化代付gas成为可能。任何人捕捉到签名并认为小费有利可图，均可代为执行交易。
而交易则通过套利形式，当任何人认为这笔交易符合自己价格预期时，均可完成交易。
定期支付。

参考实现
（非正式版，会考虑兼容20,721,1155问题；同时通过eip1271支持合约账户）
（目前研究发现，合约账户的多签缺乏标准化的接口，可能需要标准化多签合约账户的一些接口；也许需要一个新的EIP）
（基本逻辑：用户想转移一笔Token，就发起一笔签名，签名中包含转账相关信息和小费信息——即任何人帮我发起这笔转账，都可以获得小费）
ERC20
    function permitTransfer(
        address from,
        address to,
        uint256 amount,
        uint256 bounty, ///tips
        uint256 nonce,
        uint256 startTime,
        uint256 deadLine,
        bytes memory signatures
    ) external virtual{
        bytes32 digest =
            keccak256(abi.encodePacked(
                "\x19\x01",
                DOMAIN_SEPARATOR,
                keccak256(abi.encode(PERMIT_TRANSFER_TYPEHASH,
                                     from,
                                     to,
                                     amount,
                                     bounty,
                                     nonce,
                                     startTime,
                                     deadLine))
            ));
        require(block.timestamp> startTime && block.timestamp< deadLine, "invalid time");
        require(from == _recoverSigner(digest, signatures), "invalid-permitTransfer");
        require(nonce == nonces1[from][to]++, "invalid-nonce");
        _transfer(from, to, amount);
        if(bounty>0){
            _transfer(from, msg.sender, bounty);
        }
    }

    function dePermitTransfer(address to, uint256 amount) public virtual{ 
        require(amount > 0,"");
        nonces1[msg.sender][to]+= amount;
    }

+++++++++
ERC721
（基本逻辑：签名发起一笔maker，挂单；由合适的taker消耗gas吃单）
     function permitTrade (
        uint256 tokenId,
        uint256 salePrice,
        uint64 expires,
        address supportedToken,
        uint256 benchmarkPrice,
        uint256 nonce,
        bytes memory signatures
    ) external payable nonReentrant virtual{

        address tokenOwner = ownerOf(tokenId);
        address buyer = msg.sender;
        require(nonce == tradeNonces[tokenId]++, "invalid-nonce");
        ///require(nonce == nonces[tokenOwner][to]++, "invalid-nonce");

        bytes32 messageHash =
            keccak256(abi.encodePacked(
                "\x19\x01",
                DOMAIN_SEPARATOR,
                keccak256(abi.encode(PERMIT_TRADE_TYPEHASH,
                                     tokenId,
                                     salePrice,
                                     expires,
                                     supportedToken,
                                     benchmarkPrice,
                                     nonce))
        ));
        require(tokenOwner == _recoverSigner(messageHash, signatures),"");

    /// @dev Handle royalties
        (address royaltyRecipient, uint256 royalties) = _calculateRoyalties(tokenId, salePrice, benchmarkPrice);

        uint256 payment = salePrice - royalties;
        if(supportedToken == address(0)){
            require(msg.value == salePrice, "ERC6105: incorrect value");
            _processSupportedTokenPayment(royalties, buyer, royaltyRecipient, address(0));
            _processSupportedTokenPayment(payment, buyer, tokenOwner, address(0));
        }
        else{
            uint256 num = IERC20(supportedToken).allowance(buyer, address(this));
            require (num >= salePrice, "ERC6105: insufficient allowance");
            _processSupportedTokenPayment(royalties, buyer, royaltyRecipient, supportedToken);
            _processSupportedTokenPayment(payment, buyer, tokenOwner, supportedToken);
    }

        _transfer(tokenOwner, buyer, tokenId);
    ///emit Purchased(tokenId, tokenOwner, buyer, salePrice, supportedToken, royalties);
    }

    function dePermitTrade(uint256 tokenId) public virtual{ ///delist(uint256 tokenId, uint256 amount)
        require(msg.sender == ownerOf(tokenId),"");
        tradeNonces[tokenId]++;
    }
