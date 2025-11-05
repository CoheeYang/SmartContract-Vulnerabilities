Math相关的问题此处定义为由于calculation导致的问题

任何加减乘除导致的问题











https://solodit.cyfrin.io/issues/m-05-expiration-calculation-overflows-if-call-option-duration-195-days-code4rena-cally-cally-contest-git


```solidity
//这个会revert
function function1() external view returns (uint256) {
uint8 x = 128;
uint256 y = x * 2;
}
//这个不会revert，100000拿了uint16的空间
function function2() external view returns (uint256) {
uint8 x = 128;
uint256 y = x * 100000;
}

//连续的乘法中依旧有该问题，x*2先进行计算就会出现revert
//如果反过来是y*2*x，则不会revert
function function3() external view returns(uint256){
    uint8 x=128;
    uint16 y = 2;
    uint256 z = x*2*y;
}

//global varible中
//wei相关的和时间相关的days/seconds都是字面量，适用上述规则
function function4() external view returns (uint256) {
    uint8 x=128;
    uint256 y= x *2 wei;
}
//对于block中的变量，其值均为uin256，则不会出现如此问题
function function5() external view returns (uint256) {
    uint8 x=128;
    uint256 y= x *block.number;
   return block.number;
}

//非字面量会拿最大的那个空间，uint8.max *uint256变uint256不会出现revert
function ff(uint256 y) external pure returns(uint256)
{uint8 x = 128;
uint256 z = x*y;
return z;
}

```









