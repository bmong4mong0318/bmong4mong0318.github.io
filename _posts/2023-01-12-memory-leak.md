---
layout: post
title:  "메모리 누수에 관하여"
---

free는 언제 해주어야 할까?

# 메모리 누수(Memory leak)

메모리 누수란, 필요치 않은 메모리가 계속해서 점유하고 있는 현상을 말합니다. 개발자의 실수로 메모리 할당 후 해제해주지 않으면 메모리 누수가 발생합니다.

프로그램 구동 중 계속해서 메모리가 누수된다면, 아주 작은 크기의 메모리 누수라도 몇 시간, 며칠, 몇달, 몇년 이상으로 누적되면 메모리 누수는 심각해집니다. 메모리 점유율이 100%에 육박하면 윈도우가 전체적으로 느려지고 다운된 것처럼 버벅거리는 현상이 발생하게 됩니다. 양산 라인에 적용된 장비 PC에서 이런 이슈가 발생하여 PC를 재부팅해야 하는 상황이 온다면 정말 끔찍할 것입니다. 따라서 메모리 누수는 꼭 잡아주어야 합니다.

# 메모리 누수가 발생하는 경우
(내용)

# 메모리 누수의 발생

42서울 과제를 하던 중 메모리 누수가 발생하였습니다. 인자를 받아서 인자의 유효성을 검증하는 부분이였습니다.

<br>

시스템 함수를 이용하여 누수가 나는 부분을 찾아냈습니다.
```c
system("leaks push_swap | grep leaked");
```
<br>

아래 코드 부분에서 누수가 발생하였습니다. 아무리 생각해보아도 어디서 누수가 나는지 몰랐습니다.
```c
void	ft_validate_param(char *argv, t_stack *stack)
{
	char	**split_argv;
	int		i;

	i = -1;
	split_argv = ft_split(argv, ' ');
	if (!split_argv || !*split_argv)
		ft_error();
	while (*split_argv)
	{
		if (!is_nbr(*split_argv))
			ft_error();
		if (!is_integer(ft_atoi(*split_argv)))
			ft_error();
		if (!is_duplicate(ft_atoi(*split_argv), stack))
			ft_error();
		ft_push_bottom(stack, ft_init_new_node(ft_atoi(*split_argv)));
		split_argv++;
	}
}
```

<br>

우선 `ft_validate_param()` 을 호출하는 부분이 문제가 되는 것을 발견했습니다. `generate_stack()` 함수에서 while문을 통해서 `ft_validate_param()`를 반복적으로 호출하게 됩니다. 이때마다 split_argv 변수는 ft_split 내부의 malloc 함수를 통해서 힙 영역의 메모리를 할당받게 됩니다. 새롭게 메모리를 할당 받을때마다 이전에 가리키고 있었던 메모리를 잃어버리게 되는 것입니다. 따라서 `ft_validate_param()`이 종료되기전에 명시적으로 메모리 해제를 해주어야 합니다.

```c
void	generate_stack(char **argv, t_stack *stack)
{
	int	idx;

	idx = 0;
	while (argv[++idx])
	{
		if (argv[idx][0] == '\0')
			ft_error();
		else
			ft_validate_param(argv[idx], stack);
	}
}
```

<br>

## 1차 수정
```c
void	ft_validate_param(char *argv, t_stack *stack)
{
	char	**split_argv;
	int		i;

	i = 0;
	split_argv = ft_split(argv, ' ');
	if (!split_argv || !*split_argv)
		ft_error();
	while (*split_argv)
	{
		if (!is_nbr(*split_argv))
			ft_error();
		if (!is_integer(ft_atoi(*split_argv)))
			ft_error();
		if (!is_duplicate(ft_atoi(*split_argv), stack))
			ft_error();
		ft_push_bottom(stack, ft_init_new_node(ft_atoi(*split_argv)));
		split_argv++;
	}
	while (split_argv[i])
		i++;
	ft_free(split_argv, i);
}
```

그런데 이경우 또 다른 에러가 발생합니다. 첫번째 while문 안에서 split_argv의 인자이동을 위해 포인터연산을 통해서 주소를 한칸씩 옮겨주고 있습니다. while문이 끝났을때, split_argv 포인터는 split된 문자들이 아닌 정체 모를(fr_split함수를 통해 힙의 메모리를 할당받지 않은) 값의 주소를 가리키고 있을 것입니다. 이 주소를 곧바로 free 하려고 하다보니 malloc 되지 않은 메모리를 해제하려 한다는 오류가 뜨고 맙니다. 


<br>

## 2차 수정
```c
void	ft_validate_param(char *argv, t_stack *stack)
{
	char	**split_argv;
	int		i;

	i = -1;
	split_argv = ft_split(argv, ' ');
	if (!split_argv || !*split_argv)
		ft_error();
	while (split_argv[++i])
	{
		if (!is_nbr(split_argv[i]))
			ft_error();
		if (!is_integer(ft_atoi(split_argv[i])))
			ft_error();
		if (!is_duplicate(ft_atoi(split_argv[i]), stack))
			ft_error();
		ft_push_bottom(stack, ft_init_new_node(ft_atoi(split_argv[i])));
	}
	ft_free(split_argv, i);
}
```
최종적으로 while을 포인터로 순회하는 것이 아닌 인덱스로 순회하도록하여 포인터로 인해 생기는 문제점을 제거하였습니다.