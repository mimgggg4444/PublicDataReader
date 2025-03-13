``` python
def get_data(self, service, function, translate=True, **kwargs):
    """
    데이터 조회 (예외 처리 추가)
    """
    page = 1
    dataframes = []

    # 페이지 순회
    while True:
        # 메타 정보 파싱
        try:
            service = service.replace(" ", "")
            function = function.replace(" ", "")
            _service = self.meta_dict[service]['서비스']
            _function = self.meta_dict[service]['기능'][function]
            url = f"{self.endpoint}/{_service}/{_function}"
        except KeyError:
            raise ValueError(f"🚨 서비스명('{service}') 또는 기능명('{function}')이 올바르지 않습니다.")

        # 입력 파라미터 업데이트
        params = {
            "serviceKey": requests.utils.unquote(self.service_key),
            "numOfRows": self.numOfRows,
            "pageNo": page,
        }
        params.update(kwargs)
        # API 요청
        try:
            response = requests.get(url, headers=self.headers, params=params, verify=False, timeout=10)
            response.raise_for_status()
        except requests.exceptions.Timeout:
            raise Exception("⏳ 요청 시간이 초과되었습니다. 네트워크 상태를 확인하세요.")
        except requests.exceptions.ConnectionError:
            raise Exception("🔌 네트워크 연결 오류가 발생했습니다. 인터넷 연결을 확인하세요.")
        except requests.exceptions.HTTPError as http_err:
            if response.status_code == 403:
                raise Exception("🚫 API 키가 올바르지 않거나 권한이 부족합니다.")
            elif response.status_code == 429:
                raise Exception("⚠️ API 호출 제한 초과. 일정 시간 후 다시 시도하세요.")
            elif response.status_code == 500:
                raise Exception("🔧 공공데이터포털 서버 오류. 잠시 후 다시 시도하세요.")
            else:
                raise Exception(f"⚠️ API 요청 실패 (HTTP {response.status_code}): {response.text}")
        except requests.exceptions.RequestException as e:
            raise Exception(f"🚨 알 수 없는 네트워크 오류 발생: {e}")

        try:
            data = xmltodict.parse(response.text)
            body = data['response']['body']
        except (KeyError, TypeError):
            raise Exception("📄 API 응답 데이터 구조가 예상과 다릅니다. API 응답 내용을 확인하세요.")
        except Exception as e:
            raise Exception(f"📄 XML 데이터 파싱 중 오류 발생: {e}")
        
        # 응답 바디에 items 존재 시 - 데이터 프레임 행 추가
        if 'items' in body:
            items = body['items']

            # items의 값 부재 시 반복문 Break
            if items is None:
                break
            # item의 값 존재 시
            item_keys = ["bidDateInfoItem", "estimationInfo", "registered", "bidInfo", 
                         "bidHistoryInfo", "stockholderInfo", "corporatebodyInfo", "rentalInfo"]
            for key in item_keys:
                if key in items:
                    dataframes.append(self._to_dataframe(items[key]))
            if 'item' in items:
                dataframes.append(self._to_dataframe(items['item']))
        

        # 응답 바디에 item 존재 시 - 데이터 프레임 행 추가, 반복문 Break (pageNo 이슈)
        elif 'item' in body:
            dataframes.append(self._to_dataframe(body['item']))
            break
        
        # 응답 바디에 items, item 부재 시 - 반복문 Break
        else:
            break

        # 예외 - (pageNo 이슈)
        if function in ['공고공매일정', '정부재산정보공개정보상세', '캠코관리재산정보공개정보상세']:
            break

        # 페이지 번호 추가
        page += 1

    # 데이터 프레임 병합
    if dataframes:
        df = pd.concat(dataframes, ignore_index=True)
    else:
        df = pd.DataFrame()
    
    # 컬럼명 한글화
    if translate:
        df = self._translate(df, service, function)
    return df

```
