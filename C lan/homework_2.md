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


```
int main(){
	int i,j,flag=0;
	int sum=0;
	
	for(i=50;i<=100;i++){
		for(j=2;j<i/2;j++){
			if(i%j==0){
				flag=1;
				break;
			}
		}
		if(flag!=1){
			printf("%d ",i);
			sum+=i;
		}
		flag=0;
	}
	printf("\n\n%d",sum);
}

```
```
int main(){
	int i,j,sum=0;
	for(i=50;i<=100;i++){
		for(j=2;j<=i;j++)
			if(i%j==0){
				break;
			}
			
			if(i==j)
				sum+=i;
	}
	printf("%d",sum);
}
```

```
int main(){
	int i,j;
	for(i=50;i>0;i--){
		if(i%2==0 || i%3==0)
			printf("00 ");
		else
			printf("%d ",i);
		
		if(i%5==1)
			puts("\n");
	}
}
```
