```

int main(){
	int i;
	for(i=1;i<27;i+=5){
		printf("%d ",i);
	}
}
```

```

int main(){
	int i,num1,num2;
	for(i=1;i<=20;i++){
		if(i%5==0){
			num1+=i;
		}
	}
	for(i=21;i<=40;i++){
		if(i%9==0){
			num2+=i;
		}
	}
	printf("SUM:%d",num2+num1);
}
```
